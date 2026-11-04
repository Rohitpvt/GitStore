# GitStore — Security Architecture & Threat Model

**Document Version:** 1.11.0  
**Phase:** Phase 0.1.11 — Fan-Out Generation Fencing, Idempotent Retry Accounting & Schema Truthfulness  
**Status:** IN_REVIEW — Awaiting Explicit User Approval  

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

## 9. Comprehensive 19-Row Verification Matrix

In accordance with Phase 0.1.11 specifications, all 19 critical architecture controls are classified across the standard columns:

| Control | Architecture status | Evidence | Runtime status | Remaining action |
|---|---|---|---|---|
| **Global fan-out generation fence** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Mini-chunk transaction reads `/system_config/deployment_state` at same consistency boundary; aborts with `STALE_GLOBAL_GENERATION_ABORT` if `genSeq < latestPublishedSequence`) | `PENDING_INTEGRATION_TEST` | Test on Firestore emulator with interleaved generation sequences in Phase 2. |
| **Equal-sequence conflicting ID** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Explicit `HALT_CONFLICT` check when sequence matches but generation IDs differ, failing closed both globally and per-submission) | `PENDING_INTEGRATION_TEST` | Test conflicting generation sequence collision handling on emulator in Phase 2. |
| **Stale generation after newer publication** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Global deployment state check inside mini-chunk transaction prevents older generation from modifying unreached records once newer generation publishes) | `PENDING_INTEGRATION_TEST` | Test older generation resumption after newer generation publishes in Phase 4. |
| **Post-capture reapproval** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Per-submission moderation revision fence: `approvedCatalogRevision <= capturedRevision`; submissions modified/re-approved after capture are safely skipped) | `PENDING_INTEGRATION_TEST` | Test re-approval race condition during live deployment in Phase 4. |
| **Product delisting eligibility** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Transaction reads `/products/{productId}`; skips fan-out activation if product status is `delisted` or `flagged`) | `PENDING_INTEGRATION_TEST` | Test product delisting during publication in Phase 2. |
| **Transaction callback purity** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Zero outer variable mutations or logging inside callback; returns outcome object, caller accumulates only after commit succeeds) | `PENDING_INTEGRATION_TEST` | Test transaction contention retries with artificial conflict injection in Phase 2. |
| **Accurate fan-out counters** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Counts accumulated only post-commit; parent summary counters updated atomically via single completion transaction) | `PENDING_INTEGRATION_TEST` | Verify parent summary counters match sum of child batch counts in Phase 4. |
| **Deterministic batch producer** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`prepareFanoutBatches` deduplicates, validates, sorts lexicographically, chunks into max 400 IDs, and verifies child docs) | `PENDING_INTEGRATION_TEST` | Test batch generation with 1,000 synthetic submission IDs in Phase 4. |
| **Partial batch creation recovery** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Batch producer uses idempotent `set(..., { merge: true })` and full verification read-back, resuming seamlessly on worker restart) | `PENDING_INTEGRATION_TEST` | Test simulated worker interruption during batch creation in Phase 4. |
| **Fan-out preparation-complete marker** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Parent generation persists `fanoutPreparationStatus: 'prepared'` only after verification; Tier 1 publication strictly blocked if pending) | `PENDING_INTEGRATION_TEST` | Verify Tier 1 publication aborts if preparation marker is missing in Phase 4. |
| **Atomic batch completion and parent counters** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Child batch status `completed` and parent summary increments committed in a single atomic `db.runTransaction`) | `PENDING_INTEGRATION_TEST` | Test crash simulation before/after completion commit in Phase 4. |
| **Child batch manual-review transition** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Retry exhaustion or fatal conflict transitions child batch to `manual_review` and parent to `manual_review` together, logging to `/admin_logs`) | `PENDING_INTEGRATION_TEST` | Test 5-retry exhaustion routing to manual review in Phase 4. |
| **Bounded batch pagination** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Cursor pagination with `.limit(10)`, `.orderBy('batchIndex', 'asc')`, and `.startAfter(lastBatchDoc)`) | `PENDING_INTEGRATION_TEST` | Test cursor pagination across 50 batches on emulator in Phase 4. |
| **Batch concurrency ownership** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Root GitHub Actions concurrency group `github-pages` plus atomic 15-minute lease CAS check on batch documents) | `PENDING_INTEGRATION_TEST` | Test overlapping workflow lease rejection in Phase 4. |
| **SemVer validation claims** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore Rules enforce bounded string pattern; trusted worker authoritatively parses SemVer 2.0.0 specification) | `PENDING_INTEGRATION_TEST` | Test SemVer parsing edge cases in worker test suite in Phase 2. |
| **SPDX validation claims** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore Rules enforce bounded string length; trusted worker authoritatively matches against official SPDX License List) | `PENDING_INTEGRATION_TEST` | Test invalid license string rejection in worker in Phase 2. |
| **Duplicate platform handling** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore Rules enforce `arePlatformsUnique` index comparison for 1-6 items; worker verifies exact set uniqueness) | `PENDING_INTEGRATION_TEST` | Test duplicate platform rejection in Rules on emulator in Phase 2. |
| **Repository and asset relationship** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Rules enforce `repoUrl == 'https://github.com/' + repoOwner + '/' + repoName`; worker verifies assets via GitHub Releases API) | `PENDING_INTEGRATION_TEST` | Test spoofed downloadUrl rejection via GitHub API mock in Phase 2. |
| **Document metadata consistency** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (All 11 project and documentation headers synchronized to Version 1.11.0, Phase 0.1.11 `IN_REVIEW`) | `PENDING_INTEGRATION_TEST` | Automated documentation header validation script in CI in Phase 2. |
