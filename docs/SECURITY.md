# GitStore — Security Architecture & Threat Model

**Document Version:** 1.13.0  
**Phase:** Phase 0.1.13 — Work-Plan Concurrency, Lease Renewal & Reconciliation Correctness  
**Status:** IN_REVIEW — Awaiting Explicit User Approval  
**Data Schema Version:** 1.13.0  

---

## 1. Security Principles & Architecture Boundary

GitStore operates under a zero-trust model regarding client browsers. Because the application frontend is hosted statically on GitHub Pages and runs entirely on the client side, **the client runtime is treated as potentially adversarial.**

All authorization gates, role escalations, state transitions, and file verifications are governed either at the database layer (Cloud Firestore Security Rules) or within an authenticated, isolated server environment (GitHub Actions Runners).

> [!IMPORTANT]
> **Evidence Classification Standard:** To preserve rigorous engineering integrity, security controls are classified strictly by their verified evidence state (`DESIGNED`, `DOCUMENTATION_VERIFIED`, `CONFIGURATION_VERIFIED`, `IMPLEMENTED`, `INTEGRATION_TESTED`, or `PENDING_INTEGRATION_TEST`). Controls are never described as "enforced" or "tested" at runtime until live deployed verification has occurred.

---

## 2. Separation of Operational Actors & Privilege Matrix

| Operation | Public Visitors | Authenticated Developers | Administrators | GitHub Actions Worker (`firebase-admin`) |
|---|---|---|---|---|
| **Browse Static Storefront** | Read `data/catalog.json` | Read `data/catalog.json` | Read `data/catalog.json` | Generates & deploys static files |
| **Read Active Products** | Public read (Firestore) | Public read (Firestore) | Public read (Firestore) | Superuser Read/Write |
| **Read User Documents (`/users`)** | ❌ Forbidden | ✅ Own document only | ❌ Forbidden (client) | Superuser Read/Write |
| **Read Public Profiles (`/public_profiles`)** | Public read | Public read | Public read | Superuser Read/Write |
| **Submit New Application** | ❌ Forbidden | ✅ Create `/submissions` (`status: queued`, payload only) | ✅ Create `/submissions` | Ingests from GitHub Issues |
| **Edit Existing Submission** | ❌ Forbidden | ✅ Own submissions only (if `queued`/`rejected`) | ❌ Enqueues via `moderation_requests` | Superuser Read/Write |
| **Submit Claim Request** | ❌ Forbidden | ✅ Create `/claim_requests` (`status: pending`) | ✅ Create `/claim_requests` | Superuser Read/Write |
| **Execute Ownership Claim** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Authoritative Admin SDK transaction |
| **Submit Abuse Report** | ❌ Forbidden | ✅ Create `/reports` (`status: open`) | ✅ Create `/reports` | Superuser Read/Write |
| **Claim Processing Lease** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Transactional superuser claim |
| **Verify Repository & Asset**| ❌ Forbidden | ❌ Client assertion only | ❌ Client assertion only | ✅ Authoritative GitHub REST API checks |
| **Approve / Publish Product** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues with Action-Bound Proof | ✅ Authoritative state transition |
| **Grant Developer Verification** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues with Action-Bound Proof | ✅ Sets `isVerifiedDeveloper: true` |
| **Consume Step-Up Nonce** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Transactional write to `/consumed_stepup_nonces` |
| **Write to `admin_logs`** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Immutable append-only write |
| **Delist Live Product** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues with Action-Bound Proof | ✅ Transitions product to `delisted` |
| **Read Internal Collections** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden (client) | Superuser Read/Write |

---

## 3. Audited Administrator Security, Root Bootstrap & Action-Bound Step-Up Proof

### 3.1 The Canonical GitStore Repository Boundary Check
Administrator authorization is verified strictly against **GitStore's own canonical repository** (`Rohitpvt/GitStore`).
To prevent privilege escalation from forks, the worker explicitly validates its execution context:
```javascript
if (process.env.GITHUB_REPOSITORY !== 'Rohitpvt/GitStore') {
  throw new Error(`SECURITY ALERT: Execution on non-canonical repository '${process.env.GITHUB_REPOSITORY}' rejected.`);
}
```
Repository environment strings alone are not sufficient protection against untrusted workflow execution. Privileged workflows are strictly isolated to trusted default-branch events, and fork PR code execution is completely prohibited (Section 4).

### 3.2 Action-Bound Step-Up Proof, Nonce Ledger & ActionHash Classification (Option A)
To eliminate vulnerabilities stemming from long-lived browser sessions, session hijacking, credential theft, or unauthorized actions performed via linked passwordless email accounts, GitStore enforces **Mandatory Fresh GitHub Step-Up Reauthentication with Action-Bound Proofs and Nonce Replay Protection (Option A)**:

1. **Interactive Step-Up Challenge:**
   When an administrative action is triggered in `admin.html`, the client initiates an interactive reauthentication popup:
   ```javascript
   import { getAuth, GithubAuthProvider, reauthenticateWithPopup } from 'firebase/auth';

   const auth = getAuth();
   const provider = new GithubAuthProvider();
   provider.addScope('read:user');

   // Interactive popup challenges admin to re-authenticate at GitHub
   const result = await reauthenticateWithPopup(auth.currentUser, provider);
   const tokenResult = await auth.currentUser.getIdTokenResult(true);
   ```

2. **Elimination of Raw Token Storage & ActionHash Classification:**
   **Zero raw OAuth access tokens are written to or stored in Cloud Firestore.**
   The client constructs an action-bound proof:
   ```javascript
   const actionNonce = crypto.randomUUID().replace(/-/g, '') + crypto.randomUUID().replace(/-/g, '');
   const authTime = tokenResult.claims.auth_time;
   const targetDocumentPath = `submissions/${targetSubmissionId}`;
   
   // Compute SHA-256 over action, target path, nonce, and auth_time
   const rawMessage = `${action}:${targetDocumentPath}:${actionNonce}:${authTime}`;
   const actionHash = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(rawMessage))
     .then(b => Array.from(new Uint8Array(b)).map(x => x.toString(16).padStart(2, '0')).join(''));

   const stepUpProof = {
     providerId: 'github.com',
     authTime: authTime,
     actionNonce: actionNonce,
     targetDocumentPath: targetDocumentPath,
     actionHash: actionHash
   };
   ```
   > [!IMPORTANT]
   > **Truthful Cryptographic Classification:** The client-generated SHA-256 `actionHash` contains no server secret, private signing key, or HMAC key. It is **not** an independent digital signature or authentication proof. It functions strictly as a **client data-integrity checksum** ensuring that the tuple `(action, targetDocumentPath, actionNonce, authTime)` was not corrupted or modified in transit. The true authorization foundation relies upon the verified Firebase token claims, GitHub identity matching, collaborator permissions on `Rohitpvt/GitStore`, and atomic worker-side nonce consumption.

3. **Atomic Nonce Reservation in `/consumed_stepup_nonces`:**
   To prevent an attacker or malicious script from reusing an unexpired nonce across *different* request documents, the worker consumes nonces in an atomic Firestore transaction:
   - When leasing a request, the worker computes `nonceHash = SHA256(actionNonce)`.
   - The worker reads `/consumed_stepup_nonces/{nonceHash}` inside the transaction.
   - If the document exists, the transaction aborts with `NONCE_ALREADY_CONSUMED`.
   - If not, the worker records `{ nonceHash, requestId, requestedByUid: req.requestedByUid, action: req.action, targetDocumentPath: req.targetDocumentPath, workerRunId, consumedAt }` and transitions the request to `processing`.
   - If privileged processing fails downstream, the nonce **remains permanently burned**, requiring the administrator to perform fresh interactive reauthentication.
   - Client access to `/consumed_stepup_nonces` is default-deny (`allow read, write: if false;`).

4. **Action-Specific Moderation Payload Schemas & Target Path Validation:**
   The worker and Firestore Security Rules validate that `targetDocumentPath` matches canonical targets derived from `action` and `targetId`. Furthermore, `payload` is validated against action-specific nested schemas (Section 3.2 of [`docs/DATABASE_SCHEMA.md`](./DATABASE_SCHEMA.md)), rejecting arbitrary nested keys or unapproved properties.

---

## 4. GitHub Actions Workflow Security & Fork Threat Model

### 4.1 Strict Trigger Constraints & Secrets Withholding
- `schedule` and `workflow_dispatch` are the **only permitted privileged triggers**.
- `pull_request_target` and `workflow_run` triggers are **prohibited** to eliminate fork secret-exfiltration attacks.
- Standard `pull_request` runs on untrusted fork code with zero secret access.

### 4.2 Actions Runner Environment & Node 24 Action Runtime Compatibility
GitHub removed Node 20 from hosted Actions runners on September 23, 2026. All actions are pinned to official Node 24-compatible releases using 40-character commit SHAs:
- `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1` (`runs.using: 'node24'`, requires runner `>= v2.327.1`)
- `actions/setup-node@0a44ba7841725637a54e280ec667c276c6cc814e # v7.1.0` (`runs.using: 'node24'`, requires runner `>= v2.327.1`)
- `actions/upload-pages-artifact@56afc609e74202658d3ffba0e8f6dda462b719fa # v5.0.0` (`runs.using: 'composite'`, packages gzip/tar)
- `actions/deploy-pages@d6db90164ac5ed86f2b6aed7e0febac5b3c0c03e # v5.0.1` (`runs.using: 'node24'`, requires runner `>= v2.327.1`)

**Native Git CLI Credential Behavior:** Official documentation confirms that `actions/checkout` v7.0.1 securely isolates stored credentials in `$RUNNER_TEMP` (preventing accidental repository artifact leaks) while configuring the Git credential helper to permit native Git commands (`git fetch`, `git push origin main`) to execute seamlessly when `persist-credentials: true` (default) with `permissions: contents: write`.

### 4.3 Exhaustive Nested `productData` Schema Validation & Authoritative Worker Verification
Validation is strictly separated between client-facing Firestore Security Rules and server-side trusted worker execution:
- **Structural Rules Enforcement (`isValidProductData`):**
  - **Basic Info:** `name` (3-60 chars), `slug` (canonical regex `^[a-z0-9]+(-[a-z0-9]+)*$`, 3-30 chars), `repoUrl` (GitHub HTTPS URL <= 200 chars), `repoOwner` (1-39 chars), `repoName` (1-100 chars), `releaseTag` (1-50 chars), `shortDescription` (10-200 chars), `fullDescription` (20-10000 chars), `category` (strict enum).
  - **Repository Consistency:** Strict assertion `p.repoUrl == 'https://github.com/' + p.repoOwner + '/' + p.repoName` directly in Security Rules.
  - **Version Format:** Bounded string in Rules (1-32 chars, regex `^[0-9A-Za-z.+-]+$`).
  - **License Format:** Bounded string in Rules (1-50 chars).
  - **Selected Asset:** Map allowlist `['name', 'extension', 'sizeBytes', 'downloadUrl', 'assetId']`. All 5 mandatory. `sizeBytes` (1 to 524,288,000 bytes = 500 MiB). `downloadUrl` (GitHub release download URL pattern). `assetId` (positive integer).
  - **Media:** `iconUrl` (optional HTTPS URL <= 500 chars), `screenshots` list (max 6 items, each HTTPS URL <= 500 chars).
  - **Platforms:** `supportedPlatforms` list (1-6 items, `hasOnly(['windows', 'macos', 'linux', 'android', 'ios', 'web'])`, plus explicit uniqueness check via `arePlatformsUnique`).
- **Authoritative Trusted Worker Verification:**
  - **SemVer 2.0.0 Validation:** The trusted worker authoritatively parses `version` against the full SemVer 2.0.0 specification (`MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]`), rejecting malformed versions.
  - **SPDX License List Verification:** The trusted worker validates `license` against the official SPDX License List catalog (e.g. MIT, Apache-2.0, GPL-3.0-only, BSD-3-Clause, etc.), rejecting arbitrary strings.
  - **Set Uniqueness:** The worker validates `new Set(supportedPlatforms).size === supportedPlatforms.length`.
  - **Release Asset Proof:** The worker queries the GitHub REST API (`GET /repos/{owner}/{repo}/releases/tags/{tag}`) to authoritatively verify that `selectedAsset.assetId` and `selectedAsset.downloadUrl` are genuine GitHub Release assets belonging to the declared repository, preventing client URL spoofing.

---

## 5. Repository Ownership & Inherent Release Authority

### 5.1 Release Authority Model
GitStore enforces a strict tier-based ownership model:
1. **Direct Personal Repository Owners (`repo.owner.id == submitterGithubId`):** Possess inherent release authority. Submissions passing automated asset validation are eligible for automated publication.
2. **Organization Members & Committers:** Possess commit rights but not proven platform release authorization. All such submissions are gated to `needs_review` for administrator approval.
3. **Community Nominators:** Third-party submissions are treated as unverified suggestions and require mandatory review.

---

## 6. Firebase App Check & Free-Tier Operational Limits

### 6.1 Provider Specification: reCAPTCHA Enterprise
- ReCAPTCHA Enterprise provides **10,000 free assessments per month** across the Google Cloud Organization.
- Initialized with `isTokenAutoRefreshEnabled: true` on authenticated developer pages only.
- Public storefront visitors load static catalog snapshots directly from GitHub Pages and consume **0 App Check assessments** and **0 Firestore reads**.

---

## 7. Least-Privilege Split-Job Pages Architecture & Clean Staging

### 7.1 Split-Job Structure & Verified Permissions
1. **`build-and-commit`:** Granted `contents: write`. Read access to repository metadata is granted implicitly by GitHub Actions by default; invalid `metadata: read` is purged. Compiles static catalogs, constructs `site/`, asserts `stagedDigest === catalogDigest`, executes git commit only if `commit_needed == 'true'`, exports outputs to `$GITHUB_OUTPUT`, and uploads Pages artifact (`path: site/`).
2. **`deploy-to-pages`:** Granted `pages: write`, `id-token: write`, `contents: read` targeting `environment: github-pages`. Runs conditionally when `needs.build-and-commit.outputs.deploy_needed == 'true'`. Evaluates pre-deploy ordering guard, dispatches deployment via `actions/deploy-pages@v5.0.1`, verifies dedicated Pages status (`succeed`), and executes 2-tier transactional finalization with durable fan-out recovery.

### 7.2 Staging Provenance & Shell Injection Defense
- `site/` staging is constructed cleanly from the exact target commit and verified via recomputed catalog digest (`stagedDigest = computeCatalogDigest('site/data')`).
- Commit SHAs are validated against `/^[0-9a-f]{40}$/i` before shell execution.
- Output bridge writes to `$GITHUB_OUTPUT` reject values containing newline delimiters to prevent workflow command injection.

---

## 8. Software Safety Disclaimer & Legal Non-Execution

GitStore operates under an absolute non-execution invariant:
> "GitStore provides discovery and metadata verification only. GitStore does NOT execute, sandbox, or certify compiled binaries. Downloads are served directly from the author's public GitHub repository. Verify software source and run appropriate local anti-malware scanners prior to running executables."

---

## 9. Verification & Architecture Control Matrices

### 9.1 Architecture Security Controls Classification

In accordance with GitStore governance specifications, critical architecture controls are classified across standardized evidence columns:

| Control | Architecture status | Evidence | Runtime status | Remaining action |
|---|---|---|---|---|
| **Global fan-out generation fence** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Mini-chunk transaction reads `/system_config/deployment_state` at same consistency boundary; aborts with `STALE_GLOBAL_GENERATION_ABORT` if `genSeq < latestPublishedSequence`) | `PENDING_INTEGRATION_TEST` | Test on Firestore emulator with interleaved generation sequences in Phase 2. |
| **Equal-sequence conflicting ID** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Explicit `HALT_CONFLICT` check when sequence matches but generation IDs differ, failing closed both globally and per-submission) | `PENDING_INTEGRATION_TEST` | Test conflicting generation sequence collision handling on emulator in Phase 2. |
| **Stale generation after newer publication** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Global deployment state check inside mini-chunk transaction prevents older generation from modifying unreached records once newer generation publishes) | `PENDING_INTEGRATION_TEST` | Test older generation resumption after newer generation publishes in Phase 4. |
| **Post-capture reapproval fence** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Per-submission moderation revision fence: positive integer check, `approvedCatalogRevision <= capturedRevision`; submissions re-approved after capture are skipped with `skipped_post_capture_reapproval`) | `PENDING_INTEGRATION_TEST` | Test re-approval race condition during live deployment in Phase 4. |
| **Fail-closed product status check** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Transaction reads `/products/{productId}`; missing doc fails closed as `skipped_product_not_found`; `delisted`/`flagged` items fail closed and are never reactivated or published) | `PENDING_INTEGRATION_TEST` | Test product delisting / deletion race during fan-out in Phase 2. |
| **Concurrency-safe preparation lock** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Transaction `acquirePreparationLock` claims 15-minute ownership on `/system_deployments/{generationId}`; concurrent worker rejected with `PREPARATION_LOCKED_BY_OTHER_WORKER`; expired lock reclaimable) | `PENDING_INTEGRATION_TEST` | Test concurrent runner collision on same generation in Phase 4. |
| **Mandatory workPlanDigest validation** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Canonical 64-char hex format `/^[0-9a-f]{64}$/` strictly validated; missing or malformed digest fails closed as `PREPARED_DIGEST_MISSING_OR_INVALID_ABORT`; blocks Tier 1 publication) | `PENDING_INTEGRATION_TEST` | Test malformed/missing digest rejection in Phase 2. |
| **Protected active batch contents** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Reads existing batch during preparation; actively leased batch with differing plan throws `ACTIVE_BATCH_MUTATION_CONFLICT`; completed batch is strictly immutable with `COMPLETED_BATCH_CORRUPT_CONFLICT`) | `PENDING_INTEGRATION_TEST` | Test preparation retry while batch is actively leased in Phase 4. |
| **Bounded cleanup & verification** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Stale batch cleanup and deep readback verification use bounded cursor pagination `limit(50)` on `batchIndex` ASC; never deletes completed or actively leased batches) | `PENDING_INTEGRATION_TEST` | Test multi-page subcollection verification on emulator in Phase 2. |
| **Option A heartbeat lease renewal** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Transactional `renewBatchLease` triggered when remaining lease < 5m; asserts `leaseOwner === workerExecutionId`; enforces 60m ceiling from `leaseAcquiredAt`; aborts with `LEASE_RENEWAL_LOST_ABORT`) | `PENDING_INTEGRATION_TEST` | Test long-running batch renewal and superseded worker rejection in Phase 4. |
| **Atomic manual-review audit logging** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Single atomic Firestore transaction reads all docs before writes, transitions child batch to `manual_review`, parent to `manual_review`, and writes deterministic audit doc `fanout_mr_{genId}_{batchId}_attempt_{attempts}`) | `PENDING_INTEGRATION_TEST` | Test retry idempotency and audit preservation under emulator contention in Phase 2. |
| **Stable immutable reconciler pagination** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Startup reconciler orders cursor query strictly on immutable `generationSequence` ASC with `limit(10)` and `startAfter(lastGenDoc)`; status field mutations during traversal do not shift cursor position) | `PENDING_INTEGRATION_TEST` | Test multi-generation sweep with concurrent status mutations in Phase 4. |

---

### 9.2 18-Scenario Formal Verification Matrix

The following test matrix specifies initial states, operations, expected outcomes, document mutation boundaries, and recovery behaviors for all 18 critical architectural scenarios identified in Phase 0.1.13:

> [!NOTE]
> **Specification vs. Execution Status:** All 18 scenarios below represent formal **Documented Test Specifications** (`DOCUMENTED_TEST_SPECIFICATION`). In accordance with Phase 0.1 non-negotiable constraints, zero application code is executed during Phase 0.1. All scenarios are assigned runtime status `PENDING_INTEGRATION_TEST` for automated execution against the Firebase Emulator Suite in Phase 2 and Phase 4.

| # | Scenario Name | Initial State | Operation | Expected Result | Documents Allowed to Change | Documents That Must Remain Unchanged | Recovery Behavior | Status |
|---|---|---|---|---|---|---|---|---|
| **1** | **Two workers preparing different plans concurrently** | Parent generation doc `gen_001` exists with `fanoutPreparationStatus: 'pending'`. Worker 1 has work plan A (`["sub_001", "sub_002"]`, digest `aaa...`); Worker 2 has conflicting work plan B (`["sub_001", "sub_003"]`, digest `bbb...`). | Worker 1 and Worker 2 invoke `prepareFanoutBatches` concurrently. | Worker 1 executes `acquirePreparationLock` transaction first, setting `fanoutPreparationStatus: 'preparing'`, `preparationOwner: worker1`, `preparationExpiresAt: now + 15m`. Worker 2's transaction detects active lock held by Worker 1 (`status === 'locked'`) and throws `PREPARATION_LOCKED_BY_OTHER_WORKER`. Worker 2 halts without writing batch documents. | `/system_deployments/gen_001` (Worker 1 lock), child batches for Worker 1. | All submissions, global deployment state, Worker 2 batches (not created). | Worker 2 exits cleanly; Worker 1 completes preparation. If Worker 1 crashes, lock expires after 15m, allowing subsequent worker to safely reclaim. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **2** | **Prepared generation with missing digest** | `gen_002` has `status: 'deploying'`, `fanoutPreparationStatus: 'prepared'`, but `workPlanDigest: null` or malformed (not 64-char hex). Live manifest matches Pages CDN. | Reconciler executes Tier 1 publication transaction. | Inside transaction, pre-write assertion checks `workPlanDigest` against `/^[0-9a-f]{64}$/`; detects missing/invalid digest; throws `PREPARED_DIGEST_MISSING_OR_INVALID_ABORT`. Transaction rolls back with zero writes. Reconciler catches error and routes generation to `manual_review`. | `/system_deployments/gen_002` (`status: 'failed'`, `reconciliationStatus: 'manual_review'`) | `/system_config/catalog_state`, `/system_config/deployment_state`, `/submissions/*`. | Publication strictly blocked. Admin audits corrupt deployment record via `/admin.html` and re-triggers clean compilation. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **3** | **Prepared generation with corrupted child batch** | `gen_003` has `fanoutPreparationStatus: 'prepared'` and valid `workPlanDigest`. However, child batch `batch_00000` was externally corrupted (digest mismatch or missing IDs). | Idempotent preparation retry invoked, or worker re-verifies work plan. | `prepareFanoutBatches` detects `already_prepared`, invokes `verifyPersistedBatches`. Paginated readback compares `bData.batchDigest` with expected canonical digest; detects corruption; throws `FANOUT_PREPARATION_VERIFICATION_FAILED`. | None | `/system_deployments/gen_003`, `/system_config/deployment_state`, `/submissions/*`. | Fails closed. Prevents corrupt child batch from propagating. Parent is not republished; worker/admin restarts preparation to re-create clean child batch. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **4** | **Stale generation snapshot during preparation** | Caller passes stale in-memory `genData` with `fanoutPreparationStatus: 'pending'` to `prepareFanoutBatches`, but Firestore already transitioned `gen_004` to `preparing` or `prepared`. | `prepareFanoutBatches` executes. | Function ignores passed-in `genData` and executes `acquirePreparationLock` transaction which fetches fresh `genDocRef`. Detects live state is already `preparing` (by another worker) or `prepared`; returns lock rejection or verifies existing batches without overwriting. | None (if already locked) or child batch verification. | `/system_deployments/gen_004` (not clobbered by stale snapshot). | Stale memory snapshot cannot overwrite live database state; concurrency safety preserved. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **5** | **Active lease during a preparation retry** | `gen_005` preparation is retried while Worker A holds an active processing lease on `batch_00000` (`status: 'processing'`, `leaseExpiresAt > now`, `leaseOwner: workerA`). Stored work plan perfectly matches expected plan. | Retry worker executes step 6 in `prepareFanoutBatches`. | Retry worker inspects `existingSnap`. Detects active lease; verifies `existing.batchDigest === batchDigest` and `existing.submissionIds` matches `chunkIds`. Content matches perfectly; retry worker leaves active batch untouched (`continue`), preserving lease metadata and avoiding clobbering active worker. | None (batch untouched) | `batch_00000` (`leaseOwner`, `leaseExpiresAt`, `status` preserved). | Active Worker A finishes uninterrupted; retry worker completes verification safely. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **6** | **Attempt to mutate completed batch contents** | `gen_006` child batch `batch_00000` is already `status === 'completed'` with digest `abc...`. A buggy or malicious retry attempts to prepare batch 0 with conflicting IDs (digest `xyz...`). | `prepareFanoutBatches` inspects existing batch 0. | Detects `status === 'completed'`. Compares existing digest `abc...` with expected digest `xyz...`. Digest mismatch detected; throws `COMPLETED_BATCH_CORRUPT_CONFLICT: Mutation of completed batch forbidden`. Mutation rejected immediately. | None | `batch_00000` (remains `status: 'completed'`), `/system_deployments/gen_006`. | Fails closed. Completed batches remain permanently immutable. Worker halts and alerts administrator. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **7** | **Interrupted paginated stale-batch cleanup** | `gen_007` has 120 leftover batches from an aborted run with 150 batches; current run needs only 2 batches. Cleanup executes cursor pagination (`limit(50)`). Worker crashes after deleting page 1 (50 docs). | Worker restarts and re-invokes `prepareFanoutBatches`. | Re-executes `acquirePreparationLock` (after lease expiry or by same worker). Resumes paginated cleanup from sequence 0: reads page 1 (next 50 docs), deletes unleased stale docs; reads page 2 (remaining 20 docs), deletes them; reaches end of stale collection safely. | Leftover stale unleased batch docs (deleted). | Active leased batches, completed batches, `/system_config/deployment_state`. | Cleanup is fully idempotent across pages; crash recovery safely resumes without orphaned or undeleted stale docs. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **8** | **Interrupted paginated readback verification** | `gen_008` persists 100 child batches. Readback verification processes page 1 (50 batches) successfully, then worker process is killed before verifying page 2. | Workflow restarts and re-runs `prepareFanoutBatches`. | Lock is reclaimed. Persist step checks batches (matches existing, skips re-writes). Readback verification restarts from page 1: verifies page 1 (50 batches), verifies page 2 (50 batches). Asserts total verified count (100) equals `totalBatches` (100). Atomically updates parent `fanoutPreparationStatus: 'prepared'`. | `/system_deployments/gen_008` (`fanoutPreparationStatus: 'prepared'` set ONLY after full readback). | Any incomplete or unverified batches. | Verification boundary is atomic: parent preparation marker is never persisted until 100% of batches across all pages pass verification. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **9** | **Lease renewal by the current owner** | Worker 1 owns active lease on `batch_00001` (`leaseOwner: worker1`, `leaseExpiresAt: now + 3m`, `leaseAcquiredAt: now - 10m`). Worker 1 is about to process mini-chunk 3. | Worker 1 detects remaining lease < 5m; calls `renewBatchLease(db, batchRef, worker1)`. | Transaction reads `batch_00001`. Asserts `leaseOwner === worker1` (matches). Asserts `leaseExpiresAt > now` (valid). Asserts `now - leaseAcquiredAt (10m) < 60m` (within limit). Advances `leaseExpiresAt` to `now + 15m`, sets `lastHeartbeatAt = now`. Transaction commits successfully. | `fanout_batches/batch_00001` (`leaseExpiresAt: now + 15m`, `lastHeartbeatAt: now`, `updatedAt`). | All submissions, parent generation, other batches. | Worker 1 continues processing mini-chunk 3 safely with an extended 15-minute lease window. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **10** | **Lease renewal by a superseded owner** | Worker 1 experienced a network freeze; its lease expired. Worker 2 claimed the lease (`leaseOwner: worker2`). Worker 1 unfreezes and attempts to renew the lease. | Worker 1 calls `renewBatchLease(db, batchRef, worker1)`. | Transaction reads `batchRef`. Evaluates `bData.leaseOwner` (`worker2`) !== `workerExecutionId` (`worker1`). Throws `LEASE_RENEWAL_LOST_ABORT: lease owner mismatch`. Transaction aborts with zero writes. Worker 1 catches abort and shuts down immediately. | None | `fanout_batches/batch_00001` (remains owned by Worker 2), all submissions. | Worker 1 exits immediately without writing data. Worker 2 continues execution safely without collision. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **11** | **Lease expiry during a long-running mini-chunk sequence** | Worker 1 takes longer than 60 minutes continuously processing complex batch `batch_00002` (`leaseAcquiredAt: now - 61m`). Worker attempts lease renewal. | Worker 1 calls `renewBatchLease(db, batchRef, worker1)`. | Transaction reads batch doc; detects `now - leaseAcquiredAt > 60 * 60 * 1000`. Throws `MAX_LEASE_WINDOW_EXCEEDED`. Renewal is rejected. Worker catches error, halts batch processing, and fails closed. | None | Batch document (lease not extended), uncommitted submissions. | Prevents indefinite zombie lockups. Worker halts; batch lease naturally expires, allowing fresh worker to resume remaining submissions safely. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **12** | **Worker crash during renewal** | Worker 1 initiates `renewBatchLease`; runner crashes midway while transaction is in flight before commit. | GitHub Actions runner aborts. | Firestore transaction rolls back automatically on client disconnection. `leaseExpiresAt` remains at prior timestamp. No partial writes persist. | None | `fanout_batches/batch_00003`, all submissions. | Batch retains its un-renewed lease expiry. If runner does not recover before expiry, lease expires naturally and next scheduled workflow reclaims batch. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **13** | **Manual-review transaction with atomic audit logging** | Worker encounters 5th failed attempt or fatal `HALT_CONFLICT` on `batch_00004` of `gen_009`. | Error handler executes manual-review transaction. | Single `db.runTransaction` executes: reads parent `gen_009`, reads `batch_00004`, reads audit log `fanout_mr_gen_009_batch_00004_attempt_5`. All reads precede writes. Updates batch to `status: 'manual_review'`, clears lease. Updates parent to `submissionFanoutStatus: 'manual_review'`. Writes `/admin_logs` entry with deterministic ID. Commits atomically. | `fanout_batches/batch_00004`, `system_deployments/gen_009`, `/admin_logs/fanout_mr_gen_009_batch_00004_attempt_5`. | All other batches, all submissions. | Both child and parent transition together; audit log is guaranteed written in the same commit. Silent manual-review transition without audit log is physically impossible. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **14** | **Duplicate transaction retry and audit idempotency** | Manual-review transaction encounters network glitch during commit; Firestore client library automatically retries the transaction. | Transaction callback re-executes. | Reads parent, child, and audit doc `fanout_mr_gen_009_batch_00004_attempt_5`. Audit doc exists or is about to be set with identical deterministic ID. Transaction writes updates to parent, child, and idempotent `tx.set(auditLogRef)`. Commits cleanly. | Same 3 documents as scenario 13. | Zero duplicate audit records created in `/admin_logs`. | Deterministic document ID guarantees strict audit ledger idempotency across any number of transaction retries. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **15** | **Reconciliation while status fields change** | Reconciler executes cursor pagination on `system_deployments` ordered by `generationSequence` ASC (`limit(10)`). Page 1 returns `[gen_010 (seq 1), gen_011 (seq 2)]`. During processing of `gen_010`, its status mutates from `pending` to `completed`. | Reconciler queries Page 2 using `startAfter(gen_011)`. | Because cursor ordering is strictly on immutable `generationSequence` ASC, the cursor position is anchored to `generationSequence: 2`. Mutating `gen_010`'s status does NOT shift document positions. Page 2 returns `[gen_012 (seq 3), gen_013 (seq 4)]` cleanly without skipping any generation. | Updated status on processed generations. | Cursor integrity preserved; zero skipped documents. | Eventual consistency guaranteed across traversal without record skipping. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **16** | **Multiple scheduled reconciler runs recovering remaining work** | Reconciler run 1 processes 5 of 8 incomplete generations before hitting GitHub Actions step time budget. Generations 6, 7, and 8 remain `partial`. | Scheduled GitHub Actions workflow triggers 15 minutes later. | Run 2 starts a fresh cursor traversal from sequence 0. Skips already completed generations (excluded by `where('submissionFanoutStatus', 'in', ['pending', 'in_progress', 'partial'])`). Discovers generations 6, 7, and 8 immediately. Resumes their fan-out batches to completion. | Generations 6, 7, 8 and their respective fanout batches and submissions. | Generations 1-5 (already completed, untouched). | Multi-run eventual consistency achieved without claiming false "exactly-once" guarantees in a single execution. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **17** | **README/document-index version consistency** | Root `README.md`, `docs/README.md`, and all 9 sub-specifications are inspected for versioning and link integrity. | Comprehensive cross-document consistency audit executed. | Document version is uniformly `1.13.0` across all 11 files. Data schema version is `1.13.0`. ADR range is `ADR-01 to ADR-26`. Verification matrix has exactly 18 documented scenarios. Manifest `schemaVersion` is `"1.13.0"`. All markdown links use valid paths. | All 11 documentation files synchronized. | Project history, historical ADR text. | Complete cross-document alignment verified with zero drift. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **18** | **Phase 0.1.11 governance status consistency** | Master project governance rules inspected in `PHASE_PLAN.md` and `ROADMAP.md`. | Audit of phase statuses and gating authority. | Phase 0.1.11 status is `IN_REVIEW` ⏳ (awaiting explicit user approval). Phase 0.1.12 status is `IN_REVIEW` ⏳. Phase 0.1.13 status is `IN_REVIEW` ⏳. Master Phase 0.1 is `IN_REVIEW` ⏳. Phase 1 remains strictly `NOT_STARTED` 🛑. No phase is marked `COMPLETED` or `APPROVED` without satisfying established governance gates. | `docs/PHASE_PLAN.md`, `docs/ROADMAP.md` | Phase 0 `APPROVED` baseline; Phase 1 `NOT_STARTED` boundary. | Strict governance maintained; implementation work on Phase 1 remains blocked pending user acceptance. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
