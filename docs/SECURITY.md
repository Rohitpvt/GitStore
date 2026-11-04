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

## 9. Comprehensive 17-Row Verification Matrix

In accordance with Phase 0.1.9 specifications, all 17 critical controls are classified across the 5 standard columns:

| Area | Architecture status | Evidence | Runtime status | Remaining action |
|---|---|---|---|---|
| **Valid GitHub Actions permissions** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Unsupported key `metadata: read` purged; only documented GITHUB_TOKEN keys used) | `PENDING_INTEGRATION_TEST` | Validate workflow parsing on GitHub Actions in Phase 4. |
| **Build job permission scope** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`contents: write` only; repo metadata granted implicitly by platform) | `PENDING_INTEGRATION_TEST` | Test catalog push on runner in Phase 4. |
| **Deploy job permission scope** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`pages: write`, `id-token: write`, `contents: read` strictly scoped to `github-pages` env) | `PENDING_INTEGRATION_TEST` | Verify OIDC minting and Pages deploy in Phase 4. |
| **Case B commit-needed separation** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`commit_needed` separated from `deploy_needed`; commit step skipped during Case B) | `PENDING_INTEGRATION_TEST` | Test Case B execution skipping commit step in Phase 4. |
| **Recovered SHA preservation** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Case B exports recovered commit SHA without overwriting with `git rev-parse HEAD`) | `PENDING_INTEGRATION_TEST` | Test recovered SHA consistency across split jobs in Phase 4. |
| **New-generation SHA handling** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (New generation commits `data/`, captures new SHA, validates 40-char format, and exports) | `PENDING_INTEGRATION_TEST` | Test new catalog commit creation and SHA capture in Phase 4. |
| **Cross-job provenance consistency** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Job 1 `resolve-sha` step ensures artifact and deployment metadata reference exact same commit SHA) | `PENDING_INTEGRATION_TEST` | Assert commit SHA match between jobs in Phase 4. |
| **Tier 1 publication state** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (3-write transaction advances `deployment_state` and `catalog_state`, sets `submissionFanoutStatus: 'pending'`) | `PENDING_INTEGRATION_TEST` | Test atomic state anchor on Firestore Emulator in Phase 2. |
| **Tier 2 fan-out state** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Batched updates of 400 docs track `submissionFanoutCursor` on generation doc) | `PENDING_INTEGRATION_TEST` | Test submission fan-out batch chunking in Phase 4. |
| **Partial batch recovery** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Idempotent updates and cursor tracking ensure crash between batches resumes safely) | `PENDING_INTEGRATION_TEST` | Simulate crash between batches and verify resumption in Phase 4. |
| **Fan-out discovery query** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Exact Firestore query discovers `status == 'published'` and `submissionFanoutStatus in ['pending', 'in_progress']`) | `PENDING_INTEGRATION_TEST` | Verify composite index and reconciler query in Phase 4. |
| **Expected submission ID naming** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Renamed to `expectedSubmissionIds` matching `/submissions/{id}` document IDs) | `PENDING_INTEGRATION_TEST` | Validate field name consistency in processor tests in Phase 4. |
| **Catalog digest file scope** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Canonical contract includes `catalog.json`, `categories.json`, `products/*.json`; excludes manifest, HTML, assets) | `PENDING_INTEGRATION_TEST` | Test digest invariance against HTML edits in Phase 4. |
| **Normal generation digest** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Compiler invokes `computeCatalogDigest('data')` using canonical JSON serialization) | `PENDING_INTEGRATION_TEST` | Test digest test vectors on compiled output in Phase 4. |
| **Case B digest** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Reconciler invokes `computeCatalogDigest('site/data')` asserting match with `gen.catalogDigest`) | `PENDING_INTEGRATION_TEST` | Test Case B staged digest match in Phase 4. |
| **Action metadata and nested dependency** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`upload-pages-artifact@v5.0.0` verified using `actions/upload-artifact@v7.0.1`; `checkout@v5.0.0` retained over `v6.x`) | `PENDING_INTEGRATION_TEST` | Validate composite action resolution on runner in Phase 4. |
| **Documentation-only evidence classification** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Strictly classified as DESIGNED/DOCUMENTATION_VERIFIED; zero unbuilt files claimed as live) | `PENDING_INTEGRATION_TEST` | Confirm working tree has zero application code in Phase 0.1. |
