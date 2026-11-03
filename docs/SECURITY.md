# GitStore — Security Architecture & Threat Model

**Document Version:** 1.6.0  
**Phase:** Phase 0.1.5 — Final Security Rules, Administrator Step-Up Authentication & Deployment Recovery Audit  
**Status:** Approved Security Specification (Hardened & Reconciled)  

---

## 1. Security Principles & Architecture Boundary

GitStore operates under a zero-trust model regarding client browsers. Because the application frontend is hosted statically on GitHub Pages and runs entirely on the client side, **the client runtime is treated as potentially adversarial.**

All authorization gates, role escalations, state transitions, and file verifications are governed either at the database layer (Cloud Firestore Security Rules) or within an authenticated, isolated server environment (GitHub Actions Runners).

> [!IMPORTANT]
> **Evidence Classification Standard:** To preserve rigorous engineering integrity, security controls are classified strictly by their verified evidence state (`DESIGNED`, `DOCUMENTATION_VERIFIED`, `CONFIGURATION_VERIFIED`, `IMPLEMENTED`, `INTEGRATION_TESTED`, or `PENDING_INTEGRATION_TEST`). Controls are never described as "enforced" at runtime until live deployed verification has occurred.

---

## 2. Separation of Operational Actors & Privilege Matrix

| Operation | Public Visitors | Authenticated Developers | Administrators | GitHub Actions Worker (`firebase-admin`) |
|---|---|---|---|---|
| **Browse Static Storefront** | Read `data/catalog.json` | Read `data/catalog.json` | Read `data/catalog.json` | Generates & deploys static files |
| **Read Active Products** | Public read (Firestore) | Public read (Firestore) | Public read (Firestore) | Superuser Read/Write |
| **Read User Documents (`/users`)** | ❌ Forbidden | ✅ Own document only | ❌ Forbidden (client) | Superuser Read/Write |
| **Read Public Profiles (`/public_profiles`)** | Public read | Public read | Public read | Superuser Read/Write |
| **Submit New Application** | ❌ Forbidden | ✅ Create `/submissions` (`status: queued`) | ✅ Create `/submissions` | Ingests from GitHub Issues |
| **Edit Existing Submission** | ❌ Forbidden | ✅ Own submissions only (if `queued`/`rejected`) | ❌ Enqueues via `moderation_requests` | Superuser Read/Write |
| **Submit Claim Request** | ❌ Forbidden | ✅ Create `/claim_requests` (`status: pending`) | ✅ Create `/claim_requests` | Superuser Read/Write |
| **Execute Ownership Claim** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Authoritative Admin SDK transaction |
| **Claim Processing Lease** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Transactional superuser claim |
| **Verify Repository & Asset**| ❌ Forbidden | ❌ Client assertion only | ❌ Client assertion only | ✅ Authoritative GitHub REST API checks |
| **Approve / Publish Product** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues with Fresh Step-Up Proof | ✅ Authoritative state transition |
| **Grant Developer Verification** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues with Fresh Step-Up Proof | ✅ Sets `isVerifiedDeveloper: true` |
| **Write to `admin_logs`** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Immutable append-only write |
| **Delist Live Product** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues with Fresh Step-Up Proof | ✅ Transitions product to `delisted` |
| **Read Internal Collections** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden (client) | Superuser Read/Write |

---

## 3. Audited Administrator Security, Root Bootstrap & Step-Up Reauthentication

### 3.1 The Canonical GitStore Repository Boundary Check
Administrator authorization is verified strictly against **GitStore's own canonical repository** (`Rohitpvt/GitStore`).
To prevent privilege escalation from forks, the worker explicitly validates its execution context:
```javascript
if (process.env.GITHUB_REPOSITORY !== 'Rohitpvt/GitStore') {
  throw new Error(`SECURITY ALERT: Execution on non-canonical repository '${process.env.GITHUB_REPOSITORY}' rejected.`);
}
```
Repository environment strings alone are not sufficient protection against untrusted workflow execution. Privileged workflows are strictly isolated to trusted default-branch events, and fork PR code execution is completely prohibited (Section 4).

### 3.2 Mandatory Fresh GitHub Step-Up Reauthentication (Option A)
To eliminate vulnerabilities stemming from long-lived browser sessions, session hijacking, credential theft, or unauthorized actions performed via linked passwordless email accounts, GitStore enforces **Mandatory Fresh GitHub Step-Up Reauthentication (Option A)** for all privileged operations (`APPROVE_SUBMISSION`, `REJECT_SUBMISSION`, `VERIFY_DEVELOPER`, `REVOKE_DEVELOPER`, `DELIST_PRODUCT`, `FEATURE_PRODUCT`):

1. **Interactive Step-Up Challenge:**
   When an administrative button is clicked in `admin.html`, the client initiates an interactive reauthentication popup:
   ```javascript
   import { getAuth, GithubAuthProvider, reauthenticateWithPopup } from 'firebase/auth';

   const auth = getAuth();
   const provider = new GithubAuthProvider();
   provider.addScope('read:user');

   // Interactive popup challenges admin to re-authenticate at GitHub
   const result = await reauthenticateWithPopup(auth.currentUser, provider);
   const credential = GithubAuthProvider.credentialFromResult(result);
   const ephemeralGithubToken = credential.accessToken;
   const issuedAt = new Date();
   const expiresAt = new Date(issuedAt.getTime() + 300 * 1000); // 300s expiration
   const actionNonce = crypto.randomUUID().replace(/-/g, '') + crypto.randomUUID().replace(/-/g, '');
   ```

2. **Security Rules Freshness Gate:**
   Firestore Security Rules enforce that any moderation request document write must carry a Firebase Auth token with `auth_time` within 300 seconds of `request.time`:
   ```javascript
   function isFreshlyAuthenticated() {
     return request.auth != null 
       && request.auth.token.auth_time != null
       && ((request.time.toMillis() / 1000) - request.auth.token.auth_time < 300);
   }

   match /moderation_requests/{requestId} {
     allow create: if isFreshlyAuthenticated()
       && request.resource.data.keys().hasOnly(['requestId', 'action', 'targetId', 'payload', 'requestedByUid', 'stepUpProof', 'status', 'createdAt'])
       && request.resource.data.requestId == requestId
       && request.resource.data.requestedByUid == request.auth.uid
       && request.resource.data.stepUpProof.providerId == 'github.com'
       && request.resource.data.stepUpProof.ephemeralGithubToken is string
       && request.resource.data.stepUpProof.ephemeralGithubToken.size() > 0
       && request.resource.data.status == 'pending'
       && request.resource.data.createdAt == request.time;
     allow update, delete: if false;
   }
   ```

3. **Ephemeral Step-Up Proof Verification in Batch Worker:**
   - Worker retrieves `stepUpProof` from `/moderation_requests/{requestId}`.
   - Worker validates that `Date.now() <= Date.parse(stepUpProof.expiresAt)`.
   - Worker queries GitHub's authoritative API using the ephemeral token:
     `GET https://api.github.com/user` with `Authorization: Bearer ${stepUpProof.ephemeralGithubToken}`.
   - Worker verifies that the returned numeric user ID (`ghUser.id`) exactly matches `verifiedCallerNumericId` (resolved via Firebase Admin SDK from `userRecord.providerData.find(p => p.providerId === 'github.com')`).
   - Worker verifies canonical repository collaborator permissions (`Rohitpvt/GitStore`).

4. **Immediate Token Destruction & Anti-Replay Guarantee:**
   - In the same atomic Firestore update transitioning the request to `processing`, the worker **immediately deletes `ephemeralGithubToken`**:
     ```javascript
     await db.collection('moderation_requests').doc(requestId).update({
       'stepUpProof.ephemeralGithubToken': admin.firestore.FieldValue.delete(),
       'stepUpProof.consumedAt': new Date().toISOString(),
       'stepUpProof.workerRunId': String(process.env.GITHUB_RUN_ID),
       status: 'processing'
     });
     ```
   - Zero raw tokens are retained in persistent database records.
   - Any attempt to replay a moderation request with a missing token or an existing `consumedAt` timestamp is aborted immediately.

5. **Technical Boundaries & Limitations:**
   - Firebase Auth ID tokens carry Firebase cryptographic signatures asserting claims, but do not contain arbitrary third-party cryptographic signatures from GitHub.
   - Client browsers cannot sign arbitrary payloads with GitHub credentials without an interactive OAuth flow.
   - Option A combines Firebase Auth's built-in `auth_time` freshness claim with an ephemeral, single-use GitHub OAuth access token verified directly via GitHub REST API by the worker and destroyed immediately upon consumption.

### 3.3 Root Administrator Bootstrap Verification Flow
Repository ownership alone does not authenticate the person making an administrative request:

1. **Retrieve Canonical Repository Owner Identity:**
   The worker queries GitHub's repository metadata:
   `GET https://api.github.com/repos/Rohitpvt/GitStore`
   The response contains `owner.id`—an immutable integer assigned by GitHub (e.g. `12345678`). Alternatively, this integer is pre-verified and configured in the worker environment as `BOOTSTRAP_ROOT_ADMIN_GITHUB_ID`.
2. **Compare Caller Identity Against Owner Identity:**
   ```javascript
   function normalizeGithubId(val) {
     if (val == null) return null;
     const str = String(val).trim();
     if (!/^\d+$/.test(str)) return null;
     return str;
   }

   const canonicalOwnerNumericId = normalizeGithubId(repoMetadata.owner.id);
   const isRootAdministrator = (verifiedCallerNumericId === canonicalOwnerNumericId);
   ```
3. **Fail-Closed Authorization:**
   - If `isRootAdministrator` is `true`, root administrative authority is established.
   - If `false`, the worker checks whether the caller is an authorized delegated maintainer (Section 3.4).
   - If neither, the request is rejected with `403 FORBIDDEN` and an audit entry is created in `/admin_logs`.
   *(Evidence: DESIGNED and DOCUMENTATION_VERIFIED; live comparison is PENDING_INTEGRATION_TEST).*

### 3.4 Fail-Closed Collaborator Permission Check (Delegated Maintainers)
For callers other than the root administrator, the worker resolves the current login handle via `GET https://api.github.com/user/{verifiedCallerNumericId}` and queries GitHub's collaborator permission endpoint:
`GET https://api.github.com/repos/Rohitpvt/GitStore/collaborators/{verifiedLogin}/permission`

**Endpoint Response Structure (Officially Verified):**
```json
{
  "permission": "write",
  "role_name": "maintain",
  "user": { "login": "octocat", "id": 583231 }
}
```
- `permission`: Legacy base role (`admin`, `write`, `read`, `none`). Modern `maintain` role maps to `write` under legacy `permission`.
- `role_name`: Explicit role name assigned (`admin`, `maintain`, `write`, `triage`, `read`, or custom organization roles).

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

The architecture strictly rejects relying solely on repository name string checks. Forked repositories or malicious pull requests can compromise build environments if untrusted events are granted elevated secrets.

GitStore enforces an **exhaustive event allowlist and restriction matrix** with formal evidence classifications:

| Workflow Trigger Event | Authorization Status | Secrets Access | Allowed Context & Ref Restrictions | Security Controls & Rationale | Evidence Classification |
|---|---|---|---|---|---|
| **`schedule`** | **ALLOWED** | Yes (`FIREBASE_SERVICE_ACCOUNT`, `GITHUB_TOKEN`) | Default branch (`refs/heads/main`) only | Runs the scheduled batch worker. Executes strictly on the canonical repository default branch. Completely isolated from external PR code. | `DOCUMENTATION_VERIFIED` (Workflow syntax); live execution is `PENDING_INTEGRATION_TEST` |
| **`workflow_dispatch`** | **ALLOWED** | Yes (`FIREBASE_SERVICE_ACCOUNT`, `ADMIN_VERIFY_TOKEN`) | Canonical repo collaborators with `write`/`admin` | Allows on-demand batch runs or manual catalog re-compilation. Gated to verified repository collaborators by GitHub Actions permissions. | `DOCUMENTATION_VERIFIED` (GitHub API permissions); live check is `PENDING_INTEGRATION_TEST` |
| **`issues: opened, edited`** | **RESTRICTED INGEST** | Scoped / None | Ingestion workflow only | Ingests Issue Form submissions. Runs with minimal permissions (`issues: read`, `contents: read`). Runs parser in strict isolation; never executes user-supplied code or shell commands. | `DESIGNED` |
| **`pull_request`** | **RESTRICTED** | **NO SECRETS** | Pull requests targeting `main` | Read-only static linting and validation of docs/schema. GitHub Actions automatically withholds repository secrets from fork-originated PRs. | `DOCUMENTATION_VERIFIED` (Actions runner security boundary) |
| **`pull_request_target`** | ❌ **FORBIDDEN** | **NONE (BANNED)** | Not permitted in GitStore repository | **CRITICAL:** `pull_request_target` executes in the context of the base repository with full access to secrets, but can check out and run untrusted code from the fork PR. Strictly prohibited. | `DESIGNED` (Repository policy ban) |
| **`workflow_run`** | ❌ **FORBIDDEN** | **NONE (BANNED)** | Not permitted for untrusted triggers | Avoids untrusted artifact injection and cross-workflow privilege escalation. | `DESIGNED` |
| **`repository_dispatch`** | ❌ **RESTRICTED** | Restricted | Not used in MVP | Not needed under single-workflow Option A architecture. | `DESIGNED` |
| **`push`** | **RESTRICTED** | No Privileged Pipeline | Protected `main` branch only | Pushes to `main` are restricted by branch protection rules (require review/checks). Ordinary pushes do NOT trigger the privileged publisher to avoid push-suppression conflicts and deployment thrashing. | `DOCUMENTATION_VERIFIED` (Token push suppression behavior) |

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
1. **`build-and-commit`:** Granted `contents: write`, `metadata: read`. Compiles static catalogs, generates deterministic `catalogDigest`, stages public files into `site/`, commits and pushes to `origin/main`. Has **zero** Pages deployment or OIDC token permissions.
2. **`deploy-to-pages`:** Granted `pages: write`, `id-token: write`, `contents: read`, `metadata: read`. Packages the `site/` staging directory, dispatches deployment to `environment: github-pages`, verifies public manifest, and finalizes Firestore. Has **zero** repository write permissions.

### 7.2 Verified Official Action Versions
- Repository Checkout: `actions/checkout@v7` (ESM runner credential isolation, Node 24 runtime).
- Runtime Setup: `actions/setup-node@v7` (Node 24 LTS, automated package-manager caching).
- Pages Artifact Packaging: `actions/upload-pages-artifact@v5` (latest stable Pages packaging action).
- Pages Deployment Dispatch: `actions/deploy-pages@v5` (latest stable Pages deployment dispatch).

---

## 8. Software Safety Disclaimer & Legal Non-Execution

GitStore operates under an absolute non-execution invariant:
> "GitStore provides discovery and metadata verification only. GitStore does NOT execute, sandbox, or certify compiled binaries. Downloads are served directly from the author's public GitHub repository. Verify software source and run appropriate local anti-malware scanners prior to running executables."

---

## 9. Comprehensive 20-Row Verification Matrix

In accordance with Phase 0.1.5 audit requirements, all 20 critical security, recovery, and architecture controls are classified with truthful evidence and testing statuses across 6 columns:

| # | Control / Mechanism | Design Status | Evidence Classification | Runtime Test Status | Remaining Action & Target Phase |
|---|---|---|---|---|---|
| **1** | **Actions Event Allowlist & Fork Security** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (GitHub Actions runner security model) | `PENDING_INTEGRATION_TEST` | Validate trigger gating on live canonical repository (`Rohitpvt/GitStore`) in Phase 4/6. |
| **2** | **Firebase ID-Token Revocation Verification** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firebase Admin SDK `verifyIdToken(token, true)` docs) | `PENDING_INTEGRATION_TEST` | Execute integration test in Phase 2 with simulated expired/revoked tokens. |
| **3** | **Root Administrator Bootstrap Identity Comparison** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (GitHub API `owner.id` numeric format & Firebase provider UID docs) | `PENDING_INTEGRATION_TEST` | Execute live comparison test against `Rohitpvt/GitStore` owner ID in Phase 2/4. |
| **4** | **Numeric ID Normalization (`normalizeGithubId`)** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (ECMAScript regex `/^\d+$/` & string conversion) | `PENDING_INTEGRATION_TEST` | Execute unit test suite with integer, string, whitespace, and corrupt inputs in Phase 2. |
| **5** | **Claim Request Whitelist & Field Blocking** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore rules syntax `keys().hasOnly()` and size bounds) | `PENDING_INTEGRATION_TEST` | Run automated security rules test suite against Firebase Emulator in Phase 2. |
| **6** | **Mandatory Fresh GitHub Step-Up Reauthentication (Option A)** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firebase Auth `reauthenticateWithPopup` & `auth_time` claim docs) | `PENDING_INTEGRATION_TEST` | Test interactive popup and `auth_time < 300s` rules gating in Phase 2/5. |
| **7** | **Step-Up Token Direct GitHub Verification & Destruction** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (GitHub REST API `GET /user` & `FieldValue.delete()` docs) | `PENDING_INTEGRATION_TEST` | Verify token verification, immediate deletion, and anti-replay rejection in Phase 5. |
| **8** | **Private User Document Data Lockout (`/users/{uid}`)** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore Security Rules `isOwner(uid)` read/update restrictions) | `PENDING_INTEGRATION_TEST` | Test denial of unauthenticated and third-party reads on `/users` in Phase 2. |
| **9** | **Public Profile Projection Separation (`/public_profiles`)** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore Security Rules public read & worker-only badge rules) | `PENDING_INTEGRATION_TEST` | Test public read and client modification denial of `isVerifiedDeveloper` in Phase 2. |
| **10** | **Internal Collections Default-Deny Security Rules** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore Security Rules `allow read, write: if false;`) | `PENDING_INTEGRATION_TEST` | Verify client read/write rejection on `system_deployments`, `deployment_state`, `catalog_state`, `admin_logs` in Phase 2. |
| **11** | **12-State Recoverable Deployment State Machine** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Finite state machine transition logic) | `PENDING_INTEGRATION_TEST` | Simulate all 12 states and valid/invalid transitions in Phase 4 integration tests. |
| **12** | **Full-History Git Search for Deployment Manifest** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Git CLI `log` without depth limits & GitHub Commits API pagination) | `PENDING_INTEGRATION_TEST` | Verify discovery of commits beyond 10-commit window in Phase 4 test suite. |
| **13** | **Branch Ancestry Cryptographic Assertion (`merge-base`)** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Git CLI `merge-base --is-ancestor` exit code semantics) | `PENDING_INTEGRATION_TEST` | Test assertion failure against orphaned/forked commits in Phase 4. |
| **14** | **Deterministic Catalog Content Digest (`catalogDigest`)** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Node.js `crypto` SHA-256 over canonicalized sorted JSON) | `PENDING_INTEGRATION_TEST` | Verify identical digest generation across platforms and runs in Phase 4. |
| **15** | **Missing Pages Deployment ID Recovery (Cases A–D)** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (GitHub Pages API `/pages/deployments` endpoint specifications) | `PENDING_INTEGRATION_TEST` | Simulate Cases A, B, C, and D failure scenarios in Phase 4 recovery suite. |
| **16** | **Four-Case Monotonic Generation Ordering Matrix** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Firestore monotonic integer transaction logic) | `PENDING_INTEGRATION_TEST` | Simulate stale generations and sequence conflicts in Phase 4 concurrency suite. |
| **17** | **Durable Catalog Revision Tracking (`catalog_state`)** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Monotonic revision counters & dirty state detection) | `PENDING_INTEGRATION_TEST` | Test change preservation during active compilation runs in Phase 4. |
| **18** | **Least-Privilege Split-Job Pages Workflow** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (GitHub Actions workflow permissions & job dependencies) | `PENDING_INTEGRATION_TEST` | Validate token privilege separation across `build-and-commit` and `deploy-to-pages` in Phase 4. |
| **19** | **Isolated Public Staging Directory (`site/`)** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (GitHub Pages artifact packaging path specification) | `PENDING_INTEGRATION_TEST` | Assert non-existence of `.git`, `.github`, `docs/`, `scripts/` in artifact in Phase 4. |
| **20** | **GitHub Pages Non-Commercial Policy Launch Gate** | `DESIGNED` | `DOCUMENTATION_VERIFIED` (Official published GitHub Pages limits and ToS) | `PENDING_INTEGRATION_TEST` | Obtain formal written confirmation from GitHub Support in Phase 4 Launch Gate. |

