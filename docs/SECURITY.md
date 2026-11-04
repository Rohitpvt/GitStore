# GitStore — Security Architecture & Threat Model

**Document Version:** 1.9.0  
**Phase:** Phase 0.1.9 — Workflow Validation, Recovery SHA Preservation & Durable Finalization Recovery  
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

### 4.3 Exhaustive Nested `productData` Schema Validation
Both Firestore Security Rules (`isValidProductData`) and the batch worker enforce complete structural validation:
- **Basic Info:** `name` (3-60 chars), `slug` (canonical regex `^[a-z0-9]+(-[a-z0-9]+)*$`, 3-30 chars), `repoUrl` (GitHub HTTPS URL <= 200 chars), `repoOwner` (1-39 chars), `repoName` (1-100 chars), `releaseTag` (1-50 chars), `shortDescription` (10-200 chars), `fullDescription` (20-10000 chars), `category` (strict enum), `version` (1-32 chars), `license` (1-50 chars).
- **Selected Asset:** Map keys strictly allowlisted to `['name', 'extension', 'sizeBytes', 'downloadUrl', 'assetId']`. All 5 keys mandatory. `sizeBytes` strictly bounded between 1 and 524,288,000 bytes (500 MiB). `downloadUrl` must be an HTTPS GitHub release asset URL. `assetId` must be a positive integer.
- **Media:** `iconUrl` (optional HTTPS URL <= 500 chars), `screenshots` list (max 6 items; each element validated as HTTPS URL <= 500 chars).
- **Platforms:** `supportedPlatforms` list (1-6 items; elements strictly validated against enum `['windows', 'macos', 'linux', 'android', 'ios', 'web']`).
- **Enforcement:** Enforced on `create` and `update` in Firestore Rules, preventing arbitrary nested key injection or schema corruption.

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

## 9. Comprehensive 18-Row Verification Matrix

In accordance with Phase 0.1.10 specifications, all 18 critical controls are classified across the 5 standard columns:

| Area | Design status | Evidence | Runtime status | Remaining action |
|---|---|---|---|---|
| **Stale fan-out protection** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Conditional update enforces `genData.generationSequence > (currentSub.publishedGenerationSequence \|\| 0)`) | `PENDING_INTEGRATION_TEST` | Test on Firestore emulator with interleaved generation sequences in Phase 2. |
| **Newer generation preservation** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Stale generation fan-out aborts document write when newer sequence is detected; newer state preserved) | `PENDING_INTEGRATION_TEST` | Test resumption of older generation after newer generation completes in Phase 4. |
| **Post-capture moderation changes** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Status guard requires `status in ['approved', 'published']`; never resurrects `rejected`, `delisted`, `withdrawn` submissions) | `PENDING_INTEGRATION_TEST` | Test admin rejection interleaved during deployment in Phase 4. |
| **Duplicate fan-out execution** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Idempotent transactions match `publishedGenerationSequence` and `publishedGenerationId`, returning no-op) | `PENDING_INTEGRATION_TEST` | Test concurrent duplicate worker execution in Phase 4. |
| **Parent deployment document size** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Unbounded `expectedSubmissionIds` removed; parent doc holds strictly $O(1)$ scalar counters << 1 MiB) | `PENDING_INTEGRATION_TEST` | Verify parent document size remains constant with 10k submissions in Phase 4. |
| **Bounded fan-out batch documents** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Subcollection `/system_deployments/{generationId}/fanout_batches/{batchId}` capped at 400 IDs = ~13 KB << 1 MiB) | `PENDING_INTEGRATION_TEST` | Test batch document chunking and 1 MiB threshold adherence in Phase 4. |
| **Unique expected submission IDs** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Deterministic batch partitioning deduplicates submission IDs prior to subcollection batch writes) | `PENDING_INTEGRATION_TEST` | Test deduplication during batch generation in Phase 4. |
| **Batch discovery and pagination** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Ordered query on `batchIndex asc` with composite indexes supports clean cursor-less pagination) | `PENDING_INTEGRATION_TEST` | Test batch pagination query and resumption across runner boundaries in Phase 4. |
| **401 error handling** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (HTTP 401 classified as fatal `API_AUTH_ERROR`; halts loop, retains `pending_retry`, alerts administrator) | `PENDING_INTEGRATION_TEST` | Mock HTTP 401 on Commits API and verify immediate abort in Phase 4. |
| **Network error handling** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Timeouts, connection resets, DNS failures classified as `NETWORK_ERROR`; sets `isSearchIncomplete = true`, retains `pending_retry`) | `PENDING_INTEGRATION_TEST` | Mock ECONNRESET and verify generation retained in `pending_retry` in Phase 4. |
| **Empty-search correctness** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`EMPTY_SEARCH` declared ONLY on 100% complete pagination with zero network errors, zero auth errors, zero corrupt blobs) | `PENDING_INTEGRATION_TEST` | Test exhaustive pagination with zero matches marking `abandoned` in Phase 4. |
| **Malformed-manifest handling** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (JSON SyntaxError classified as `CANDIDATE_CORRUPT_MANIFEST`; sets `hasCorruptCandidates = true`, routes to `manual_review`) | `PENDING_INTEGRATION_TEST` | Mock invalid JSON in manifest blob and verify routing to `manual_review` in Phase 4. |
| **ProductData nested field types** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore Rules `isValidProductData` enforces strict types, lengths, and regexes for all 16 fields) | `PENDING_INTEGRATION_TEST` | Test rules evaluation with invalid field types on emulator in Phase 2. |
| **Screenshot element validation** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`isValidScreenshots` checks array size <= 6 and validates each present element as HTTPS URL <= 500 chars) | `PENDING_INTEGRATION_TEST` | Test rules rejection with non-URL or non-string screenshot elements in Phase 2. |
| **Supported platform enum validation** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`isValidPlatforms` enforces list size 1-6 and `hasOnly(['windows', 'macos', 'linux', 'android', 'ios', 'web'])`) | `PENDING_INTEGRATION_TEST` | Test rules rejection of invalid platform names in Phase 2. |
| **SelectedAsset field validation** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`isValidSelectedAsset` validates all 5 keys: `name`, `extension`, `sizeBytes` <= 500 MiB, `downloadUrl`, `assetId`) | `PENDING_INTEGRATION_TEST` | Test rules rejection when missing keys or sizeBytes > 500 MiB in Phase 2. |
| **Checkout action runtime and SHA** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Official `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1` [v7.0.1], `node24`, runner `>= v2.327.1`) | `PENDING_INTEGRATION_TEST` | Test action execution on Node 24 Actions runner in Phase 4. |
| **GitHub credential persistence** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Default `persist-credentials: true` configures git credential helper; native `git push` verified functional) | `PENDING_INTEGRATION_TEST` | Test authenticated `git push origin main` in `build-and-commit` job in Phase 4. |
