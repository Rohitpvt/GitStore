# GitStore — Security Architecture & Threat Model

**Document Version:** 1.8.0  
**Phase:** Phase 0.1.7 — Replay Protection, Firestore Field Integrity & Deployment State Consistency  
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
   - If the nonce document exists, the transaction fails closed: the request is rejected with `NONCE_ALREADY_CONSUMED` and logged to `/admin_logs`.
   - If it does not exist, the worker writes the document: `{ nonceHash, requestId, requestedByUid, action, targetDocumentPath, workerRunId, consumedAt }` and transitions the request to `status: 'processing'`.
   - **Recovery Policy:** If worker processing fails post-consumption (e.g. GitHub API 500), the nonce remains permanently burned. The administrator must initiate a fresh interactive reauthentication in the UI, generating a new nonce.
   - **Client Lockout:** Security Rules enforce `allow read, write: if false;` on `/consumed_stepup_nonces`.

4. **Target Document Path Mapping & Validation:**
   The worker derives the expected canonical path from an explicit action-to-collection mapping:
   - `APPROVE_SUBMISSION`, `REJECT_SUBMISSION` -> Must match `submissions/{id}`.
   - `VERIFY_DEVELOPER`, `REVOKE_DEVELOPER` -> Must match `users/{id}` or `public_profiles/{id}`.
   - `DELIST_PRODUCT`, `FEATURE_PRODUCT` -> Must match `products/{id}`.
   The worker asserts `docData.targetDocumentPath === expectedPath`. Any attempt to target internal collections (`system_deployments`, `system_config`, `admin_logs`, `consumed_stepup_nonces`) is rejected immediately.

5. **Security Rules Freshness Gate:**
   Firestore Security Rules evaluate the `auth_time` claim within the Firebase ID token:
   ```javascript
   function isFreshlyAuthenticated() {
     return request.auth != null 
       && request.auth.token.firebase.sign_in_provider == 'github.com'
       && ('github.com' in request.auth.token.firebase.identities)
       && request.auth.token.auth_time != null
       && ((request.time.toMillis() / 1000) - request.auth.token.auth_time < 300)
       && (request.auth.token.auth_time <= (request.time.toMillis() / 1000) + 5);
   }

   match /moderation_requests/{requestId} {
     allow create: if isFreshlyAuthenticated()
       && request.resource.data.keys().hasOnly([
         'id', 'action', 'targetId', 'targetDocumentPath', 'payload',
         'requestedByUid', 'githubUserId', 'stepUpProof', 'status', 'createdAt'
       ])
       && request.resource.data.id == requestId
       && request.resource.data.requestedByUid == request.auth.uid
       && request.resource.data.githubUserId == request.auth.token.firebase.identities['github.com'][0]
       && request.resource.data.action in [
         'APPROVE_SUBMISSION', 'REJECT_SUBMISSION', 'VERIFY_DEVELOPER',
         'REVOKE_DEVELOPER', 'DELIST_PRODUCT', 'FEATURE_PRODUCT'
       ]
       && request.resource.data.stepUpProof.keys().hasOnly([
         'providerId', 'authTime', 'actionNonce', 'targetDocumentPath', 'actionHash'
       ])
       && request.resource.data.stepUpProof.providerId == 'github.com'
       && request.resource.data.stepUpProof.authTime == request.auth.token.auth_time
       && request.resource.data.stepUpProof.targetDocumentPath == request.resource.data.targetDocumentPath
       && request.resource.data.stepUpProof.actionHash.size() == 64
       && request.resource.data.status == 'pending'
       && request.resource.data.createdAt == request.time;
     allow update, delete: if false; // Strict client immutability
   }
   ```

6. **Dual Timing Model (Creation Gate vs. Worker Execution):**
   - **5-Minute Creation Window:** The freshness constraint `request.time - auth_time < 300` is enforced **only at document creation time**. This ensures the administrator authenticated within 5 minutes when placing the command in the queue.
   - **30-Minute Queue Execution Window:** Once accepted, the document cannot be modified by clients. The Actions worker processes requests within a 30-minute queue TTL (`now - doc.createdAt.toMillis() < 30 * 60 * 1000`). If older than 30 minutes, it is marked `expired`.

### 3.3 Root Administrator Bootstrap Verification Flow
1. Retrieve canonical repository owner's immutable numeric GitHub ID via `GET https://api.github.com/repos/Rohitpvt/GitStore` -> `response.data.owner.id`.
2. Extract caller's linked numeric GitHub ID from Firebase Admin SDK user record.
3. Normalize both IDs using `normalizeGithubId()` and assert exact match.
4. If matched, root administrative authority is authorized fail-closed. If not matched, fallback to delegated maintainer check.

### 3.4 Fail-Closed Collaborator Permission Check (Delegated Maintainers)
For non-root callers, the worker resolves the current login handle via `GET https://api.github.com/user/{verifiedCallerNumericId}` and queries GitHub's collaborator permission endpoint:
`GET https://api.github.com/repos/Rohitpvt/GitStore/collaborators/{verifiedLogin}/permission`

**Fail-Closed Authorization Logic:**
```javascript
const { permission, role_name } = response.data;
const isAuthorizedAdmin = (role_name === 'admin') || (role_name == null && permission === 'admin');
const isAuthorizedMaintainer = (role_name === 'maintain');

if (!isAuthorizedAdmin && !isAuthorizedMaintainer) {
  // Fail-closed: Reject ordinary write, triage, read, custom roles, and unexpected responses
  throw new Error(`FORBIDDEN: User ${verifiedLogin} role '${role_name}' (permission '${permission}') is not authorized.`);
}
```

---

## 4. GitHub Actions Event and Fork Security Matrix

| Workflow Trigger Event | Authorization Status | Secrets Access | Allowed Context & Ref Restrictions | Security Controls & Rationale | Evidence Classification |
|---|---|---|---|---|---|
| **`schedule`** | **ALLOWED** | Yes (`FIREBASE_SERVICE_ACCOUNT`, `GITHUB_TOKEN`) | Default branch (`refs/heads/main`) only | Runs scheduled batch worker on canonical repository default branch. Completely isolated from external PR code. | `DOCUMENTATION_VERIFIED` (Workflow syntax); live execution is `PENDING_INTEGRATION_TEST` |
| **`workflow_dispatch`** | **ALLOWED** | Yes (`FIREBASE_SERVICE_ACCOUNT`, `ADMIN_VERIFY_TOKEN`) | Canonical repo collaborators with `write`/`admin` | Allows on-demand batch runs or manual catalog re-compilation. Gated to verified repository collaborators by GitHub Actions permissions. | `DOCUMENTATION_VERIFIED` (GitHub API permissions); live check is `PENDING_INTEGRATION_TEST` |
| **`issues: opened, edited`** | **RESTRICTED INGEST** | Scoped / None | Ingestion workflow only | Ingests Issue Form submissions. Minimal permissions (`issues: read`, `contents: read`). Runs parser in strict isolation; never executes user-supplied code or shell commands. | `DESIGNED` |
| **`pull_request`** | **RESTRICTED** | **NO SECRETS** | Pull requests targeting `main` | Read-only static linting and validation of docs/schema. GitHub Actions automatically withholds repository secrets from fork-originated PRs. | `DOCUMENTATION_VERIFIED` (Actions runner security boundary) |
| **`pull_request_target`** | ❌ **FORBIDDEN** | **NONE (BANNED)** | Not permitted in GitStore repository | **CRITICAL:** `pull_request_target` executes in the context of the base repository with full access to secrets, but can check out and run untrusted code from the fork PR. Strictly prohibited. | `DESIGNED` (Repository policy ban) |
| **`workflow_run`** | ❌ **FORBIDDEN** | **NONE (BANNED)** | Not permitted for untrusted triggers | Avoids untrusted artifact injection and cross-workflow privilege escalation. | `DESIGNED` |
| **`push`** | **RESTRICTED** | No Privileged Pipeline | Protected `main` branch only | Pushes to `main` are restricted by branch protection rules. Ordinary pushes do NOT trigger the privileged publisher to avoid push-suppression conflicts. | `DOCUMENTATION_VERIFIED` (Token push suppression behavior) |

---

## 5. Repository Ownership, Release Authority & Verified Developer

### 5.1 Distinction of Three Security Concepts
1. **Repository Ownership:** Proves control over the GitHub repository entity (`repo.owner.id === submitterGithubId`).
2. **Release Authority:** Proves authorization to publish executable software release binaries. Direct personal owners possess inherent release authority. Organization members, committers, and third parties do **not** possess proven release authority and are held in `needs_review`.
3. **Verified Developer Identity:** A platform trust badge awarded by GitStore administrators based on formal criteria. Owning a personal repo does not confer platform-wide verified status.

### 5.2 Multi-Tier Ownership Verification Matrix

| Tier | Submitter Status | Evidence Checked via GitHub API | What it Proves | Eligible for Auto-Publish? | Action Taken |
|---|---|---|---|---|---|
| **Tier 1: Direct Personal Owner** | Personal repo owner | `repo.owner.id == submitter.githubId` | Complete administrative control of repository | **YES** (if valid assets & release) | Auto-publishes on valid asset checks |
| **Tier 2: Organization Member** | Org repo submitter | `GET /orgs/{org}/public_members/{login}` (204) | Public affiliation with org | **NO** (does not prove repo release authority) | Gated to `needs_review` |
| **Tier 3: Committer** | Contributor to repo | `GET /repos/{owner}/{repo}/commits?author={login}` | Has authored merged commits | **NO** (does not prove release authorization) | Gated to `needs_review` |
| **Tier 4: Community Nomination**| Fan / Third-party | No ownership or commit proof found | Public open-source suggestion only | **NO** (unverified submitter) | Gated to `needs_review` |

---

## 6. Firebase App Check Provider, Quotas & Free-Tier Pricing Conditions

### 6.1 Provider Specification: reCAPTCHA Enterprise
- **Recommended Provider:** **reCAPTCHA Enterprise** (`ReCaptchaEnterpriseProvider`).
- **Organization-Level Allowance:** Under Google Cloud Fraud Defense terms, reCAPTCHA Enterprise provides a free tier of **10,000 assessments per month** calculated at the **Google Cloud Organization / Billing Account level** (shared across all projects in that organization, not per project or app).

### 6.2 Token Acquisition, Token Refresh & Firestore Requests
```javascript
import { initializeAppCheck, ReCaptchaEnterpriseProvider } from 'firebase/app-check';

const appCheck = initializeAppCheck(app, {
  provider: new ReCaptchaEnterpriseProvider(RECAPTCHA_ENTERPRISE_SITE_KEY),
  isTokenAutoRefreshEnabled: true // Explicitly configured; not assumed by default
});
```
Firestore queries consume 0 additional reCAPTCHA assessments because the App Check token is verified cryptographically by the Firestore backend.

### 6.3 Free-Tier Operating Target & Billing Prerequisites
> [!IMPORTANT]
> **Free-Tier Target & Spark Plan Prerequisite:** GitStore targets operation within published free limits under the Firebase Spark Plan **strictly without an attached billing account.**
- Under the no-cost **Spark Plan** (without billing attached), exhausting the 10,000 monthly assessments causes reCAPTCHA Enterprise to return HTTP 429 (`RESOURCE_EXHAUSTED`). Token acquisition fails and subsequent unverified Firestore writes are rejected, but zero financial charges are incurred.
- If a Google Cloud billing account is linked to the project, assessments beyond 10,000 automatically incur charges ($1.00 per 1,000). GitStore relies on the Spark configuration to avoid unexpected billing. All figures are unmeasured engineering projections with stated assumptions.

---

## 7. Least-Privilege Split-Job Pages Workflow & Staging Isolation

### 7.1 Split-Job Architecture
To enforce least privilege, the GitHub Pages workflow is split into two sequential jobs:
1. **`build-and-commit`:** Granted `contents: write`, `metadata: read`. Compiles static catalogs, generates deterministic `catalogDigest`, stages public files into `site/`, commits and pushes to `origin/main`. Has **zero** Pages deployment or OIDC token permissions. Explicitly declares cross-job outputs.
2. **`deploy-to-pages`:** Granted `pages: write`, `id-token: write`, `contents: read`, `metadata: read`. Runs pre-deployment 4-case ordering guard, packages the `site/` staging directory, dispatches deployment to `environment: github-pages`, verifies public manifest, and finalizes Firestore. Has **zero** repository write permissions.

### 7.2 Action Versioning & Pinning
- Repository Checkout: `actions/checkout@v4` (Pinned commit SHA: `11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2`).
- Runtime Setup: `actions/setup-node@v4` (Pinned commit SHA: `39370e3970a6d050c480ffad4ff0ed4d3fdee5a1 # v4.1.0`).
- Pages Artifact Packaging: `actions/upload-pages-artifact@v3` (Pinned commit SHA: `56afc609e74202658d3ffba0e8f6dda462b719fa # v3.0.1`).
- Pages Deployment Dispatch: `actions/deploy-pages@v4` (Pinned commit SHA: `d6db90164ac5ed86f2b6aed7e0febac5b3c0c03e # v4.0.5`).

---

## 8. Software Safety Disclaimer & Legal Non-Execution

GitStore operates under an absolute non-execution invariant:
> "GitStore provides discovery and metadata verification only. GitStore does NOT execute, sandbox, or certify compiled binaries. Downloads are served directly from the author's public GitHub repository. Verify software source and run appropriate local anti-malware scanners prior to running executables."

---

## 9. Comprehensive 17-Row Verification Matrix

In accordance with Phase 0.1.7 audit requirements, all 17 critical architecture, security, and deployment controls are classified with truthful design, evidence, and runtime statuses across 5 standard columns:

| Area | Design status | Evidence | Runtime status | Remaining work |
|---|---|---|---|---|
| **Pages API success status** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Official Pages deployment endpoint status `succeed`) | `PENDING_INTEGRATION_TEST` | Validate live response parsing on canonical repository in Phase 4. |
| **Pages deployment reference schema** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Schema supports numeric deployment ID or string commit SHA) | `PENDING_INTEGRATION_TEST` | Verify schema storage in `/system_deployments` in Phase 4. |
| **Case B recovery handoff** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Reconciler returns `deployNeeded=true` on 404 Pages response) | `PENDING_INTEGRATION_TEST` | Simulate missing Pages deployment and test artifact packaging in Phase 4. |
| **Artifact upload and deployment** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`upload-pages-artifact@v3` and `deploy-pages@v4` workflow handoff) | `PENDING_INTEGRATION_TEST` | Test cross-job output consumption in GitHub Actions runner in Phase 4. |
| **Submission field restrictions** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Worker-managed fields excluded from client create allowlist) | `PENDING_INTEGRATION_TEST` | Run automated security rules test suite on Firebase Emulator in Phase 2. |
| **Nested product-data restrictions** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Strict nested map `keys().hasOnly()` and string size limits) | `PENDING_INTEGRATION_TEST` | Test denial of oversized payloads and invalid nested keys in Phase 2. |
| **Action nonce uniqueness** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (64-character random hex / UUIDv4 client generation) | `PENDING_INTEGRATION_TEST` | Verify uniqueness across client sessions in Phase 5. |
| **Nonce consumption transaction** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Atomic reservation in `/consumed_stepup_nonces/{nonceHash}`) | `PENDING_INTEGRATION_TEST` | Simulate concurrent nonce reuse across different requests in Phase 5. |
| **Target document mapping** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Worker derives canonical path from action-to-collection mapping) | `PENDING_INTEGRATION_TEST` | Test rejection of path mismatches and internal collection targets in Phase 5. |
| **Mixed-provider reauthentication** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (`reauthenticateWithPopup` updates `auth_time` and claims) | `PENDING_INTEGRATION_TEST` | Execute full 9-step mixed-provider test in Firebase Emulator in Phase 2. |
| **Step-up freshness** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Security Rules gate `auth_time < 300s` at creation time) | `PENDING_INTEGRATION_TEST` | Test rejection of requests with `auth_time >= 300s` in Phase 2. |
| **Catalog revision transaction** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Explicit `capturedRevision` read and monotonic advance inside transaction) | `PENDING_INTEGRATION_TEST` | Simulate simultaneous approvals and test `isDirty` calculation in Phase 4. |
| **Stale-generation finalization** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Transaction aborts on `capturedRev < currentPublishedRev`, marking `stale_aborted`) | `PENDING_INTEGRATION_TEST` | Test stale generation abort and work preservation in Phase 4. |
| **Four-case ordering** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Shared `evaluateGenerationOrdering` function applied in guard and DB) | `PENDING_INTEGRATION_TEST` | Test all 4 cases in pre-deploy guard in Phase 4. |
| **Complete commit pagination** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Traverses `Link: rel="next"` without arbitrary 5-page depth limits) | `PENDING_INTEGRATION_TEST` | Test pagination traversal beyond 500 commits in Phase 4. |
| **Recovery error classification** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Distinguishes empty search from incomplete search -> `pending_retry`) | `PENDING_INTEGRATION_TEST` | Test transient network errors during recovery in Phase 4. |
| **Documentation consistency** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Reconciled `succeed` status, `v4` actions, schemas across all 14 files) | `DOCUMENTATION_VERIFIED` | Full Git diff audit and working tree verification in Phase 0.1.7. |
