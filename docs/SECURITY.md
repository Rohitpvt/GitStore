# GitStore — Security Architecture & Threat Model

**Document Version:** 1.12.0  
**Phase:** Phase 0.1.12 — Fan-Out Work-Plan Integrity, Lease Ownership & Fail-Closed Recovery  
**Status:** IN_REVIEW — Awaiting Explicit User Approval  
**Data Schema Version:** 1.12.0  

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
| **Fail-closed product status check** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Transaction reads `/products/{productId}`; missing doc fails closed as `skipped_product_not_found`; `delisted`/`flagged` items fail closed and are never reactivated or published) | `PENDING_INTEGRATION_TEST` | Test missing product and delisted product rejection on emulator in Phase 2. |
| **Transaction callback purity** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Zero outer variable mutations or logging inside callback; returns outcome object, caller accumulates only after commit succeeds) | `PENDING_INTEGRATION_TEST` | Test transaction contention retries with artificial conflict injection in Phase 2. |
| **Accurate fan-out counters** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Counts accumulated only post-commit; parent summary counters updated atomically via single completion transaction) | `PENDING_INTEGRATION_TEST` | Verify parent summary counters match sum of child batch counts in Phase 4. |
| **Deterministic batch producer** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`prepareFanoutBatches` validates every ID, deduplicates, sorts lexicographically, chunks into max 400 IDs, computes `workPlanDigest` and `batchDigest`, and verifies child docs) | `PENDING_INTEGRATION_TEST` | Test batch generation with 1,000 synthetic submission IDs in Phase 4. |
| **Work-plan immutability guard** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`prepareFanoutBatches` asserts stored `workPlanDigest` matches recomputed digest if already marked `prepared`, throwing `WORKPLAN_MUTATION_CONFLICT` on mismatch) | `PENDING_INTEGRATION_TEST` | Test work-plan mutation rejection on emulator in Phase 2. |
| **Stale batch pruning & deep verification** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`prepareFanoutBatches` deletes batches with `batchIndex >= totalBatches` from aborted runs and performs deep element-by-element equality check) | `PENDING_INTEGRATION_TEST` | Test leftover batch pruning on emulator in Phase 4. |
| **Transactional distributed lease protocol** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`acquireBatchLease` inside Firestore transaction sets 15-minute lease with unique `workerExecutionId`; pre-chunk and completion transactions assert active ownership) | `PENDING_INTEGRATION_TEST` | Test concurrent worker lease contention and expiry on emulator in Phase 4. |
| **Atomic batch completion & parent counters** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Child batch status `completed` and parent summary increments committed in a single atomic `db.runTransaction` asserting lease ownership) | `PENDING_INTEGRATION_TEST` | Test crash simulation before/after completion commit in Phase 4. |
| **Atomic child & parent manual-review transition** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Retry exhaustion or fatal conflict transitions child batch to `manual_review` and parent to `manual_review` together in a single transaction, logging to `/admin_logs`) | `PENDING_INTEGRATION_TEST` | Test 5-retry exhaustion routing to manual review in Phase 4. |
| **Bounded reconciler pagination** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Cursor pagination with `.limit(10)`, `.orderBy('submissionFanoutStatus', 'asc')`, `.orderBy('generationSequence', 'asc')`, and `.startAfter(lastGenDoc)`) | `PENDING_INTEGRATION_TEST` | Test cursor pagination across 50 incomplete generations on emulator in Phase 4. |
| **SemVer validation claims** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore Rules enforce bounded string pattern; trusted worker authoritatively parses SemVer 2.0.0 specification) | `PENDING_INTEGRATION_TEST` | Test SemVer parsing edge cases in worker test suite in Phase 2. |
| **SPDX validation claims** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore Rules enforce bounded string length; trusted worker authoritatively matches against official SPDX License List) | `PENDING_INTEGRATION_TEST` | Test invalid license string rejection in worker in Phase 2. |
| **Duplicate platform handling** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore Rules enforce `arePlatformsUnique` index comparison for 1-6 items; worker verifies exact set uniqueness) | `PENDING_INTEGRATION_TEST` | Test duplicate platform rejection in Rules on emulator in Phase 2. |
| **Repository and asset relationship** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Rules enforce `repoUrl == 'https://github.com/' + repoOwner + '/' + repoName`; worker verifies assets via GitHub Releases API) | `PENDING_INTEGRATION_TEST` | Test spoofed downloadUrl rejection via GitHub API mock in Phase 2. |
| **Document metadata consistency** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (All project and documentation headers synchronized to Version 1.12.0, Phase 0.1.12 `IN_REVIEW`) | `PENDING_INTEGRATION_TEST` | Automated documentation header validation script in CI in Phase 2. |

---

### 9.2 20-Scenario Formal Verification Matrix

The following test matrix specifies initial states, operations, expected outcomes, document mutation boundaries, and recovery behaviors for all critical architectural edge cases identified in Phase 0.1.12:

> [!NOTE]
> **Specification vs. Execution Status:** All 20 scenarios below represent formal **Documented Test Specifications** (`DOCUMENTED_TEST_SPECIFICATION`). In accordance with Phase 0.1 non-negotiable constraints, zero application code is executed during Phase 0.1. All scenarios are assigned runtime status `PENDING_INTEGRATION_TEST` for automated execution against the Firebase Emulator Suite in Phase 2 and Phase 4.

| # | Scenario Name | Initial State | Operation | Expected Result | Documents Allowed to Change | Documents That Must Remain Unchanged | Recovery Behavior | Status |
|---|---|---|---|---|---|---|---|---|
| **1** | **Invalid submission ID during preparation** | Parent generation doc `gen_001` exists with `fanoutPreparationStatus: 'pending'`. Raw input IDs contain `["sub_valid12345", "INVALID!ID", "sub_valid67890"]`. | `prepareFanoutBatches` invoked during deployment preparation. | Fails closed with `INVALID_SUBMISSION_ID_ABORT`. Does NOT silently filter or drop the invalid ID. | None | Parent `/system_deployments/gen_001`, child batches, `/submissions/*`. | Generation remains in `pending`. Reconciler retains generation in `pending_retry`. Admin corrects input data. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **2** | **Duplicate IDs** | Parent `gen_002` in `pending`. Raw input IDs contain duplicates: `["sub_abc12345", "sub_xyz98765", "sub_abc12345"]`. | `prepareFanoutBatches` invoked. | Deduplicates via Set to 2 unique IDs; sorts lexicographically; generates canonical `workPlanDigest`; creates `batch_00000` with `expectedCount: 2`; deep readback succeeds; marks parent `prepared`. | `/system_deployments/gen_002`, `/system_deployments/gen_002/fanout_batches/batch_00000` | All other deployments and all `/submissions/*`. | Normal progression to Tier 1 publication. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **3** | **Same batch count but different stored IDs** | `gen_003` in `pending`. A previous corrupted run wrote `batch_00000` with `["sub_corrupt999"]`. Current run has `["sub_actual111"]`. | `prepareFanoutBatches` executes and reads back batch doc. | Element-by-element comparison detects mismatch (`sub_corrupt999 !== sub_actual111`) or digest mismatch; throws `FANOUT_PREPARATION_VERIFICATION_FAILED`. Parent NOT marked `prepared`. | `fanout_batches/batch_00000` | Parent `gen_003` (`fanoutPreparationStatus` remains `pending`), `/system_config/deployment_state`. | Retry overwrites corrupt batch with merge/set or worker prunes. Publication strictly blocked until verification passes. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **4** | **Unexpected extra batch document** | `gen_004` has leftover `batch_00002` from prior aborted attempt where input list was larger. Current run needs only 2 batches (`batch_00000`, `batch_00001`). | `prepareFanoutBatches` executes stale batch pruning scan. | Detects `batchIndex: 2 >= totalBatches (2)`; deletes stale `batch_00002`; persists and verifies batches 0 and 1; parent marked `prepared` with `totalFanoutBatches: 2`. | `fanout_batches/batch_00002` (deleted), `batch_00000`, `batch_00001`, `gen_004` | `/system_config/deployment_state`, `/submissions/*`. | Stale batch safely eliminated, preventing phantom batch execution during fanout. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **5** | **Interrupted preparation followed by retry** | Worker crashes mid-way after writing 3 of 5 batches. `gen_005` retains `fanoutPreparationStatus: 'pending'`. | Reconciler restarts and re-invokes `prepareFanoutBatches` on `gen_005`. | Idempotently re-evaluates IDs, prunes stale batches, updates batches with `{ merge: true }`, reads back all 5 batches, asserts digests, marks `prepared`. | `gen_005` (marked `prepared`), `fanout_batches/batch_00000`..`00004` | `/system_config/deployment_state`, `/submissions/*`. | Seamless idempotent recovery; publication proceeds only after 100% verification passes. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **6** | **Attempt to mutate a prepared work plan** | `gen_006` has `fanoutPreparationStatus: 'prepared'` with `workPlanDigest: 'aaa...'`. | Reconciler or worker calls `prepareFanoutBatches` with altered IDs producing `workPlanDigest: 'bbb...'`. | Fails closed with `WORKPLAN_MUTATION_CONFLICT`. Zero documents overwritten. | None | `gen_006`, existing `fanout_batches/*`, `/submissions/*`. | Generation work plan remains immutable. Reconciler logs anomaly; if work plan must change, a new generation sequence must be initiated. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **7** | **Missing product document** | Submission `sub_missing_prod` has `status: 'approved'`, `targetProductId: 'prod_999_nonexistent'`. Product doc does not exist. | Fan-out worker processes mini-chunk with `sub_missing_prod`. | Fail-closed check `!prodSnap.exists` triggers; logs warning, increments `chunkSkipped++`. Submission NOT marked `published`. | Batch progress counters (`skippedCount: increment(1)`). | `/submissions/sub_missing_prod` (remains `approved`), `/products/prod_999_nonexistent` (no doc created). | Item skipped; batch proceeds to completion. Parent records skipped count. Admin investigates missing product record. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **8** | **Delisted product** | Submission `sub_delisted` targets `/products/prod_100_viewer` which has `status: 'delisted'`. | Fan-out worker processes mini-chunk with `sub_delisted`. | Fail-closed check `prodData.status === 'delisted'` triggers; increments `chunkSkipped++`. Product is NEVER reactivated, and submission is NOT published. | Batch `skippedCount`. | `/products/prod_100_viewer` (status remains `delisted`), `/submissions/sub_delisted` (status remains `approved`). | Item skipped safely without reactivating delisted catalog item. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **9** | **Flagged product** | Submission `sub_flagged` targets `/products/prod_200_crypto` which has `status: 'flagged'`. | Fan-out worker processes mini-chunk with `sub_flagged`. | Fail-closed check `prodData.status === 'flagged'` triggers; increments `chunkSkipped++`. Product is NEVER published or cleared of flag. | Batch `skippedCount`. | `/products/prod_200_crypto` (status remains `flagged`), `/submissions/sub_flagged` (status remains `approved`). | Item skipped safely. Product remains suppressed from public storefront. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **10** | **Missing approval revision** | Submission `sub_no_rev` has `status: 'approved'`, but `approvedCatalogRevision` field is omitted. | Fan-out worker processes mini-chunk with `sub_no_rev`. | Fails closed (`typeof rev !== 'number'`); increments `chunkSkipped++`. Submission is NOT marked `published`. | Batch `skippedCount`. | `/submissions/sub_no_rev` (remains `approved`, no publication sequence set), `/products/*`. | Item skipped. Admin reviews submission via `/moderation_requests` to assign valid revision. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **11** | **Null or malformed approval revision** | Submission `sub_bad_rev` has `approvedCatalogRevision: null` or `"5"` (string) or `0` or `-1` or `3.14`. | Fan-out worker processes mini-chunk with `sub_bad_rev`. | Fails closed (`!Number.isInteger(rev) || rev <= 0`); increments `chunkSkipped++`. Submission NOT marked `published`. | Batch `skippedCount`. | `/submissions/sub_bad_rev`, `/products/*`. | Item skipped without modifying submission or catalog. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **12** | **Submission reapproved after generation capture** | `gen_007` captured catalog at revision 10 (`capturedRevision: 10`). `sub_reapproved` was re-approved by admin at revision 12 (`approvedCatalogRevision: 12`). | Fan-out worker for `gen_007` processes `sub_reapproved`. | Fence evaluation `approvalRev > capturedRev` triggers (`12 > 10`). Increments `chunkSkipped++`. Submission NOT updated with `gen_007` sequence. | Batch `skippedCount`. | `/submissions/sub_reapproved` (retains `approvedCatalogRevision: 12`), `/products/*`. | Item skipped cleanly for `gen_007`. Re-approved submission will be captured and published in a subsequent generation whose `capturedRevision >= 12`. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **13** | **Two workers attempting to acquire the same lease** | Batch `batch_00000` is in `status: 'pending'`. | Worker A (`run_101_1_uuidA`) and Worker B (`run_102_1_uuidB`) concurrently execute `acquireBatchLease()`. | Firestore serializes transactions. Worker A commits first (`leaseOwner: Worker A`). Worker B reads fresh state, sees active lease, and returns `{ acquired: false, reason: 'ACTIVE_LEASE_HELD_BY_ANOTHER' }`. Worker B skips cleanly. | `batch_00000` (updated by Worker A only). | All other batches. Worker B performs zero writes to `batch_00000`. | Worker A processes exclusively. Worker B moves to next available batch or exits. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **14** | **Lease expiry during processing** | Worker A holds lease on `batch_00000`, but processing pauses for $>15$ minutes (`leaseExpiresAt < now`). | Worker A attempts to process the next mini-chunk. | Mini-chunk transaction checks lease validity; detects expired lease; throws `LEASE_EXPIRED_ABORT`; transaction rolls back. | None | All submissions in mini-chunk, batch counters. | Worker A halts execution. A replacement worker can safely claim the expired lease via `acquireBatchLease()` and resume processing uncompleted mini-chunks. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **15** | **Old worker attempting completion after lease replacement** | Worker A's lease expired; Worker B acquired lease on `batch_00000`. Stalled Worker A awakens and attempts completion. | Worker A runs atomic completion transaction. | Transaction detects `freshBatchData.leaseOwner ('run_B') !== workerExecutionId ('run_A')`; throws `LEASE_LOST_ABORT`; rolls back completely. | None | `batch_00000` (retains Worker B lease), parent generation counters. | Worker A completion rejected. Worker B finishes processing and executes authoritative completion. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **16** | **Crash during atomic completion** | Worker completes all mini-chunks for `batch_00000`. Crashes immediately before or during `db.runTransaction` completion call. | Runner termination. Reconciler resumes after worker crash. | Transaction is ACID: either BOTH child batch (`completed`) and parent summary commit, or NEITHER commits. If uncommitted, batch remains in `processing` until lease expires. | If committed: child batch and parent summary counters. If rolled back: zero documents changed. | Submissions (already committed in completed mini-chunks). | Upon lease expiry (15m), replacement worker re-acquires batch. Mini-chunks evaluated idempotently (already published items are no-ops). Worker commits completion transaction cleanly. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **17** | **Exhausted retries transitioning both parent and child to manual review** | Batch `batch_00003` in generation `gen_009` encounters persistent fatal error (`fanoutAttempts >= 5`). | Worker catches error in `executeSubmissionFanout`. | Single atomic transaction transitions BOTH `batch_00003` (`status: 'manual_review'`, lease cleared, `lastError`) AND parent `gen_009` (`submissionFanoutStatus: 'manual_review'`). Appends audit entry to `/admin_logs`. | `batch_00003`, `gen_009`, `/admin_logs/*`. | All other batches in `gen_009`, `/submissions/*`, `/system_config/deployment_state`. | Reconciler excludes `manual_review` generations from auto-retry. Admin investigates `/admin_logs` and resets batch via Step-Up moderation action. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **18** | **Startup reconciliation with many incomplete generations** | 45 historical deployment generations exist with `status: 'published'` and `submissionFanoutStatus: 'pending'`. | Reconciler starts up and invokes paginated generation discovery (`reconcilePublishedState`). | Queries `system_deployments` in bounded pages of 10 (`limit(10)`), ordered by `submissionFanoutStatus` ASC, `generationSequence` ASC, using `startAfter(lastGenDoc)`. Fetches 5 successive pages. Memory consumption remains $O(1)$. Executes fanout sequentially. | Successfully processed parent generations and their child batches. | Generations outside query scope, completed historical deployments. | All incomplete generations discovered and processed without Firestore query timeouts or memory overflow. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **19** | **Cursor pagination while generation statuses change** | Reconciler paginates parent generations. While Page 1 is processing, an external action transitions a generation on Page 2 to `completed` or `manual_review`. | Reconciler queries Page 2 using `startAfter(lastGenDocFromPage1)`. | Monotonic `generationSequence` cursor and filter `submissionFanoutStatus in ['pending', 'in_progress', 'partial']` naturally exclude the updated document. Cursor progression remains valid and monotonic. No generations skipped or double-processed. | Generations processed on Page 2. | Page 1 documents already processed. | Clean pagination traversal across active incomplete deployments. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
| **20** | **Schema-version mismatch between documentation and deployment manifest** | Documentation specifies Data Schema Version `1.12.0`, but worker builds manifest with outdated `schemaVersion: "1.9.0"`. | Staging validation or reconciler verifies manifest against schema contract. | Staging verification detects mismatch (`manifest.schemaVersion ("1.9.0") !== EXPECTED_SCHEMA_VERSION ("1.12.0")`); rejects manifest with `SCHEMA_VERSION_MISMATCH`. Deployment halts before CDN artifact upload. | Generation error logs (`lastError`). | `deployment_state`, `catalog_state`, live Pages CDN files. | Worker build script updated to output synchronized `schemaVersion: "1.12.0"`. Deployment re-run succeeds. | `DOCUMENTED_TEST_SPECIFICATION` (`PENDING_INTEGRATION_TEST`) |
