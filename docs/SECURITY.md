# GitStore — Security Architecture & Threat Model

**Document Version:** 1.9.0  
**Phase:** Phase 0.1.8 — Node 24 Workflow Compatibility, Artifact Provenance & Transactional Deployment Integrity  
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
- `actions/checkout@08c6903cd8c0fde910a37f88322edcfb5dd907a8 # v5.0.0` (`runs.using: 'node24'`, requires runner `>= v2.327.1`)
- `actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0` (`runs.using: 'node24'`, requires runner `>= v2.327.1`)
- `actions/upload-pages-artifact@fc324d3547104276b827a68afc52ff2a11cc49c9 # v5.0.0` (`runs.using: 'composite'`, packages gzip/tar)
- `actions/deploy-pages@368f82528645a54fb793d4d04e342629a3f51346 # v5.0.1` (`runs.using: 'node24'`, requires runner `>= v2.327.1`)

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

### 7.1 Split-Job Structure
1. **`build-and-commit`:** Granted `contents: write`, `metadata: read`. Compiles static catalogs, constructs `site/`, asserts `stagedDigest === catalogDigest`, pushes commit to `origin/main`, exports outputs to `$GITHUB_OUTPUT`, and uploads Pages artifact (`path: site/`).
2. **`deploy-to-pages`:** Granted `pages: write`, `id-token: write`, `contents: read`, `metadata: read` targeting `environment: github-pages`. Runs conditionally when `needs.build-and-commit.outputs.deploy_needed == 'true'`. Evaluates pre-deploy ordering guard, dispatches deployment via `actions/deploy-pages@v5.0.1`, verifies dedicated Pages status (`succeed`), and executes 2-tier transactional finalization.

### 7.2 Staging Provenance & Shell Injection Defense
- `site/` staging is constructed cleanly from the exact target commit and verified via recomputed catalog digest (`stagedDigest === catalogDigest`).
- Commit SHAs are validated against `/^[0-9a-f]{40}$/i` before shell execution.
- Output bridge writes to `$GITHUB_OUTPUT` reject values containing newline delimiters to prevent workflow command injection.

---

## 8. Software Safety Disclaimer & Legal Non-Execution

GitStore operates under an absolute non-execution invariant:
> "GitStore provides discovery and metadata verification only. GitStore does NOT execute, sandbox, or certify compiled binaries. Downloads are served directly from the author's public GitHub repository. Verify software source and run appropriate local anti-malware scanners prior to running executables."

---

## 9. Comprehensive 16-Row Verification Matrix

In accordance with Phase 0.1.8 specifications, all 16 critical controls are classified across the 5 standard columns:

| Control | Design status | Documentation evidence | Runtime status | Remaining work |
|---|---|---|---|---|
| **Node 24 action compatibility** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Actions specify `runs.using: 'node24'` or composite; runner `>= v2.327.1`) | `PENDING_INTEGRATION_TEST` | Validate runner runtime execution on `ubuntu-latest` in Phase 4. |
| **Verified action commit SHAs** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`checkout@08c6903`, `setup-node@949feb2`, `upload-pages-artifact@fc324d3`, `deploy-pages@368f825`) | `PENDING_INTEGRATION_TEST` | Test workflow execution against verified SHAs in Phase 4. |
| **Case B exact source revision** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Reconciler extracts catalog assets directly from `catalogCommitSha` via git archive) | `PENDING_INTEGRATION_TEST` | Simulate Case B recovery and assert source revision purity in Phase 4. |
| **Clean isolated artifact staging** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Purges `site/` before staging; zero post-staging copies from workspace `main`) | `PENDING_INTEGRATION_TEST` | Verify staging directory isolation in workflow run in Phase 4. |
| **Staged digest verification** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Recomputes `stagedDigest` from `site/` and asserts match against `gen.catalogDigest`) | `PENDING_INTEGRATION_TEST` | Test staged digest mismatch abort in Phase 4. |
| **Explicit `$GITHUB_OUTPUT` bridge** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Direct append writes to `$GITHUB_OUTPUT` with newline injection guards) | `PENDING_INTEGRATION_TEST` | Test step output exports on Actions runner in Phase 4. |
| **Cross-job output integrity** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Job 1 declares 6 outputs; Job 2 consumes via `needs.build-and-commit.outputs`) | `PENDING_INTEGRATION_TEST` | Verify parameter propagation across split jobs in Phase 4. |
| **Finalization reads deployment state** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Transaction reads `deployment_state`, `catalog_state`, and `genDoc` before any writes) | `PENDING_INTEGRATION_TEST` | Verify pre-read transaction semantics on Firestore Emulator in Phase 2. |
| **Transactional generation ordering** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Evaluates `evaluateGenerationOrdering` inside transaction; aborts if stale) | `PENDING_INTEGRATION_TEST` | Test sequence collision and stale abort handling in Phase 4. |
| **Catalog revision monotonicity** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Asserts `capturedRevision >= latestPublishedRevision` and `<= latestRequestedRevision`) | `PENDING_INTEGRATION_TEST` | Test revision advancement in concurrent approval scenarios in Phase 4. |
| **Firestore transaction write limits** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (2-tier finalization: 3-write atomic state anchor + 400-doc batched status updates) | `PENDING_INTEGRATION_TEST` | Test finalization of generations with > 500 products in Phase 4. |
| **Action-specific moderation payload** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Security rules and worker validate custom schemas for all 6 moderation actions) | `PENDING_INTEGRATION_TEST` | Test rejection of invalid nested payload keys on Firebase Emulator in Phase 2. |
| **Target ID/path relationship** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Rules assert `targetDocumentPath` matches canonical collection and exact `targetId`) | `PENDING_INTEGRATION_TEST` | Test target redirection rejection on Firebase Emulator in Phase 2. |
| **Nonce ledger schema consistency** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Uses `requestedByUid` canonically across schema, rules, and transaction code) | `PENDING_INTEGRATION_TEST` | Test atomic nonce reservation and replay rejection in Phase 5. |
| **Commit content API error handling** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Classifies 404 vs rate limits/5xx; only declares `EMPTY_SEARCH` on full clean search) | `PENDING_INTEGRATION_TEST` | Test transient network error retention as `pending_retry` in Phase 4. |
| **Pages API status handling** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Dedicated endpoint `pages/deployments/{id}` checks `status === 'succeed'`) | `PENDING_INTEGRATION_TEST` | Validate live Pages deployment status query on GitHub API in Phase 4. |
