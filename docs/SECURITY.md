# GitStore — Security Architecture & Threat Model

**Document Version:** 1.4.0  
**Phase:** Phase 0.1.3 — Deployment Integrity, Administrator Bootstrap & Workflow Security  
**Status:** Approved Security Specification (Hardened)  

---

## 1. Security Principles & Architecture Boundary

GitStore operates under a zero-trust model regarding client browsers. Because the application frontend is hosted statically on GitHub Pages and runs entirely on the client side, **the client runtime is treated as potentially adversarial.**

All authorization gates, role escalations, state transitions, and file verifications are enforced either cryptographically at the database layer (Cloud Firestore Security Rules) or within an authenticated, isolated server environment (GitHub Actions Runners).

---

## 2. Separation of Operational Actors & Privilege Matrix

| Operation | Public Visitors | Authenticated Developers | Administrators | GitHub Actions Worker (`firebase-admin`) |
|---|---|---|---|---|
| **Browse Static Storefront** | Read `data/catalog.json` | Read `data/catalog.json` | Read `data/catalog.json` | Generates & deploys static files |
| **Read Active Products** | Public read (Firestore) | Public read (Firestore) | Public read (Firestore) | Superuser Read/Write |
| **Submit New Application** | ❌ Forbidden | ✅ Create `/submissions` (`status: queued`) | ✅ Create `/submissions` | Ingests from GitHub Issues |
| **Edit Existing Submission** | ❌ Forbidden | ✅ Own submissions only (if `queued`/`rejected`) | ❌ Enqueues via `moderation_requests` | Superuser Read/Write |
| **Submit Claim Request** | ❌ Forbidden | ✅ Create `/claim_requests` (`status: pending`) | ✅ Create `/claim_requests` | Superuser Read/Write |
| **Execute Ownership Claim** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Authoritative Admin SDK transaction |
| **Claim Processing Lease** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Transactional superuser claim |
| **Verify Repository & Asset**| ❌ Forbidden | ❌ Client assertion only | ❌ Client assertion only | ✅ Authoritative GitHub REST API checks |
| **Approve / Publish Product** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues via `moderation_requests` | ✅ Authoritative state transition |
| **Grant Developer Verification** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues via `moderation_requests` | ✅ Sets `isVerifiedDeveloper: true` |
| **Write to `admin_logs`** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Immutable append-only write |
| **Delist Live Product** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues via `moderation_requests` | ✅ Transitions product to `delisted` |

---

## 3. Audited Administrator Security & Root Bootstrap Workflow

### 3.1 The Canonical GitStore Repository Boundary Check
Administrator authorization is verified strictly against **GitStore's own canonical repository** (`Rohitpvt/GitStore`).
To prevent privilege escalation from forks, the worker explicitly validates its execution context:
```javascript
if (process.env.GITHUB_REPOSITORY !== 'Rohitpvt/GitStore') {
  throw new Error(`SECURITY ALERT: Execution on non-canonical repository '${process.env.GITHUB_REPOSITORY}' rejected.`);
}
```
However, repository environment strings alone are not sufficient protection against untrusted workflow execution. As detailed in Section 4, privileged workflows are strictly isolated to trusted default-branch events, and fork PR code execution is completely prohibited.

### 3.2 Cryptographic Identity Extraction & Token Validation (Zero Browser Trust)
Browser-submitted usernames, claims, and client-asserted UIDs are completely ignored:
1. The administrator authenticates in the browser via GitHub OAuth and writes to `/moderation_requests/{requestId}` with `requestedByUid: request.auth.uid`.
2. For direct invocations or API commands, the caller provides their Firebase ID token in the Authorization header. The worker calls the Firebase Admin SDK to cryptographically verify the token signature and assert revocation state:
   ```javascript
   // Verify ID token with revocation check enabled
   const decodedToken = await admin.auth().verifyIdToken(idToken, true);
   const verifiedUid = decodedToken.uid;
   ```
3. The worker queries the Firebase Admin SDK for user record metadata:
   ```javascript
   const userRecord = await admin.auth().getUser(verifiedUid);
   const ghProvider = userRecord.providerData.find(p => p.providerId === 'github.com');
   if (!ghProvider || !ghProvider.uid) {
     throw new Error("UNAUTHORIZED: Caller has no verified GitHub identity linked in Firebase Auth");
   }
   const verifiedCallerNumericId = ghProvider.uid; // Immutable numeric GitHub ID string
   ```
4. The worker **fails closed**: if the GitHub provider is missing, unlinked, or ambiguous, all elevated access is immediately aborted.

### 3.3 Root Administrator Bootstrap Verification Flow
Repository ownership alone does not authenticate the person making an administrative request. A secure, immutable bootstrap mechanism is enforced:

1. **Retrieve Canonical Repository Owner Identity:**
   The worker queries GitHub's repository metadata:
   `GET https://api.github.com/repos/Rohitpvt/GitStore`
   The response contains `owner.id`—an immutable integer assigned by GitHub (e.g. `12345678`). Alternatively, this integer is pre-verified and configured in the worker environment as `BOOTSTRAP_ROOT_ADMIN_GITHUB_ID`.
2. **Compare Caller Identity Against Owner Identity:**
   ```javascript
   const canonicalOwnerNumericId = String(repoMetadata.owner.id);
   const isRootAdministrator = (String(verifiedCallerNumericId) === canonicalOwnerNumericId);
   ```
3. **Fail-Closed Authorization:**
   - If `isRootAdministrator` is `true`, root administrative authority is established.
   - If `false`, the worker checks whether the caller is an authorized delegated maintainer (Section 3.4).
   - If neither, the request is rejected with `403 FORBIDDEN` and an audit entry is created in `/admin_logs`.
4. **Account Unlinking & Provider Changes:**
   If the administrator unlinks their GitHub account or alters their sign-in provider in Firebase Auth, `providerData.find(...)` returns `undefined`, immediately terminating administrator access on the next worker cycle.
5. **Emergency Administrator Revocation & Account Recovery:**
   - *GitHub Account Compromise:* The owner or organization revokes repository collaborator privileges on `Rohitpvt/GitStore` in GitHub settings.
   - *Firebase Token Invalidation:* The root administrator executes token revocation via Firebase Admin SDK: `await admin.auth().revokeRefreshTokens(compromisedUid)`.
   - *Real-Time Enforcement:* Because the worker validates collaborator permissions live against GitHub on each batch cycle, revoking access on GitHub immediately halts worker execution for that identity.

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

**Token Permissions & Integration Status:**
Calling `/repos/{owner}/{repo}/collaborators/{username}/permission` requires `Metadata: read` permissions. In personal repositories, GitHub Actions tokens may behave differently from organization repositories. The worker implements a dual-token strategy:
- Primary: attempts `secrets.GITHUB_TOKEN` with `permissions: { contents: read, metadata: read }`.
- Fallback: uses `secrets.ADMIN_VERIFY_TOKEN` (a fine-grained PAT with `Metadata: read` stored in repository secrets).
*Classification: PENDING INTEGRATION TEST.*

### 3.5 Audit Logging Schema
Every privileged operation executed by the batch worker generates an immutable entry in `/admin_logs/{logId}`:
```typescript
interface AdminAuditLog {
  id: string;                        // "log_{timestamp}_{nanoid}"
  actorUid: string;                  // Firebase Auth UID
  actorGithubNumericId: string;      // Immutable numeric GitHub ID e.g. "583231"
  actorGithubLogin: string;          // Resolved login handle
  action: ModerationAction;
  targetId: string;
  authorizationResult: 'AUTHORIZED' | 'REJECTED';
  rejectionReason: string | null;
  workerRunId: string;               // GitHub Actions run ID
  timestamp: FirebaseFirestore.Timestamp;
}
```

---

## 4. GitHub Actions Event and Fork Security Matrix

The architecture strictly rejects relying solely on repository name string checks. Forked repositories or malicious pull requests can compromise build environments if untrusted events are granted elevated secrets.

GitStore enforces an **exhaustive event allowlist and restriction matrix**:

| Workflow Trigger Event | Authorization Status | Secrets Access | Allowed Context & Ref Restrictions | Security Controls & Rationale |
|---|---|---|---|---|
| **`schedule`** | **ALLOWED** | Yes (`FIREBASE_SERVICE_ACCOUNT`, `GITHUB_TOKEN`) | Default branch (`refs/heads/main`) only | Runs the scheduled batch worker. Executes strictly on the canonical repository default branch. Completely isolated from external PR code. |
| **`workflow_dispatch`** | **ALLOWED** | Yes (`FIREBASE_SERVICE_ACCOUNT`, `ADMIN_VERIFY_TOKEN`) | Canonical repo collaborators with `write`/`admin` | Allows on-demand batch runs or manual catalog re-compilation. Gated to verified repository collaborators by GitHub Actions permissions. |
| **`issues: opened, edited`** | **RESTRICTED INGEST** | Scoped / None | Ingestion workflow only | Ingests Issue Form submissions. Runs with minimal permissions (`issues: read`, `contents: read`). Runs parser in strict isolation; never executes user-supplied code or shell commands. |
| **`pull_request`** | **RESTRICTED** | **NO SECRETS** | Pull requests targeting `main` | Read-only static linting and validation of docs/schema. GitHub Actions automatically withholds repository secrets from fork-originated PRs. |
| **`pull_request_target`** | ❌ **FORBIDDEN** | **NONE (BANNED)** | Not permitted in GitStore repository | **CRITICAL:** `pull_request_target` executes in the context of the base repository with full access to secrets, but can check out and run untrusted code from the fork PR. Strictly prohibited. |
| **`workflow_run`** | ❌ **FORBIDDEN** | **NONE (BANNED)** | Not permitted for untrusted triggers | Avoids untrusted artifact injection and cross-workflow privilege escalation. |
| **`repository_dispatch`** | ❌ **RESTRICTED** | Restricted | Not used in MVP | Not needed under single-workflow Option A architecture. |
| **`push`** | **RESTRICTED** | No Privileged Pipeline | Protected `main` branch only | Pushes to `main` are restricted by branch protection rules (require review/checks). Ordinary pushes do NOT trigger the privileged publisher to avoid push-suppression conflicts and deployment thrashing. |

#### Architectural Guardrails Against Fork Escalation:
1. **Zero Untrusted Pull Request Secrets:** No workflow responding to `pull_request` receives `FIREBASE_SERVICE_ACCOUNT` or write tokens.
2. **Execution Context Verification:** All privileged workflows assert three independent conditions at startup:
   ```javascript
   if (github.repository !== 'Rohitpvt/GitStore' || 
       !['schedule', 'workflow_dispatch'].includes(github.event_name) ||
       github.ref !== 'refs/heads/main') {
     throw new Error(`SECURITY VIOLATION: Execution in untrusted context (${github.repository}, ${github.event_name}, ${github.ref}) aborted.`);
   }
   ```
3. **Single Trusted Workflow Run (Option A):** Catalog generation, Git commit, Pages artifact upload, Pages deployment, and Firestore finalization run sequentially in a single trusted execution run, preventing cross-run race conditions or token handoff vulnerabilities.

---

---

## 5. Repository Ownership, Release Authority & Verified Developer

### 5.1 Distinction of Three Security Concepts
1. **Repository Ownership:** Proves control over the GitHub repository entity (`repo.owner.id === submitterGithubId`).
2. **Release Authority:** Proves authorization to publish executable software release binaries. Direct personal owners possess inherent release authority. Organization members, committers, and third parties do **not** possess proven release authority and are held in `needs_review`.
3. **Verified Developer Identity:** A platform trust badge awarded by GitStore administrators based on formal criteria (e.g. established open-source track record, GPG-signed commits, or manual review).
> [!IMPORTANT]
> A repository owner does **NOT** automatically receive a "Verified Developer" badge merely because they own the repository. Owning a personal repo permits auto-publication of that repo's valid releases, but does not confer platform-wide verified status.

### 5.2 Multi-Tier Ownership Verification Matrix

| Tier | Submitter Status | Evidence Checked via GitHub API | What it Proves | Eligible for Auto-Publish? | Action Taken |
|---|---|---|---|---|---|
| **Tier 1: Direct Personal Owner** | Personal repo owner | `repo.owner.id == submitter.githubId` | Complete administrative control of repository | **YES** (if valid assets & release) | Auto-publishes on valid asset checks |
| **Tier 2: Organization Member** | Org repo submitter | `GET /orgs/{org}/public_members/{login}` (204) | Public affiliation with org | **NO** (does not prove repo release authority) | Gated to `needs_review` |
| **Tier 3: Committer** | Contributor to repo | `GET /repos/{owner}/{repo}/commits?author={login}` | Has authored merged commits | **NO** (does not prove release authorization) | Gated to `needs_review` |
| **Tier 4: Community Nomination**| Fan / Third-party | No ownership or commit proof found | Public open-source suggestion only | **NO** (unverified submitter) | Gated to `needs_review` |

### 5.3 Secure Claiming Workflow & Hardened Field Whitelisting (`/claim_requests`)
To claim GitHub Issue Form submissions without creating fake accounts or exposing database ownership fields:
1. Issue ingestion records `submitterGithubId` (numeric string), `claimStatus: 'unclaimed'`, and `developerUid: null`.
2. Developer authenticates in Firebase Auth.
3. Developer submits a claim request to `/claim_requests/{requestId}`:
   ```typescript
   {
     requestId: string;
     requesterUid: string;          // Must equal request.auth.uid
     targetSubmissionId: string;
     status: 'pending';
     createdAt: serverTimestamp();
   }
   ```
4. **Strict Client Field Whitelisting via Security Rules:**
   Clients must **never** be able to inject worker-managed or privileged fields. Firestore Security Rules enforce an exhaustive whitelist:
   ```javascript
   match /claim_requests/{requestId} {
     allow read: if request.auth != null && resource.data.requesterUid == request.auth.uid;
     allow create: if request.auth != null
       && request.resource.data.keys().hasOnly(['requestId', 'requesterUid', 'targetSubmissionId', 'status', 'createdAt'])
       && request.resource.data.requestId == requestId
       && request.resource.data.requesterUid == request.auth.uid
       && request.resource.data.targetSubmissionId is string
       && request.resource.data.targetSubmissionId.size() > 0
       && request.resource.data.targetSubmissionId.size() <= 128
       && request.resource.data.status == 'pending'
       && request.resource.data.createdAt == request.time;
     allow update, delete: if false; // Privileged batch worker only via Admin SDK
   }
   ```
   **Explicitly Blocked Privileged Fields:** Requests attempting to inject `developerUid`, `verifiedGithubId`, `claimStatus`, `isAdmin`, `processedAt`, `reviewedAt`, `reviewedBy`, `resolution`, `rejectionReason`, `workerLease`, or `workerId` are rejected by the rule engine.
5. **Worker Transaction Requirements:**
   - Worker queries Firebase Admin SDK: `admin.auth().getUser(claimReq.requesterUid)`.
   - Locates `userRecord.providerData.find(p => p.providerId === 'github.com')`. Fail closed if absent.
   - Extracts `verifiedGithubId = provider.uid`.
   - Verifies target submission: `submission.submitterGithubId === verifiedGithubId`.
   - Transactionally asserts `submission.developerUid == null` (preventing race conditions or competing claims).
   - Sets `submission.developerUid = claimReq.requesterUid`, `submission.claimStatus = 'claimed'`, `claimReq.status = 'completed'`.
   - Appends audit log to `/admin_logs`. Duplicate requests are handled idempotently.

---

## 6. Firebase App Check Provider, Quotas & Billing Configuration

### 6.1 Provider Specification: reCAPTCHA Enterprise
- **Recommended Provider:** **reCAPTCHA Enterprise** (`ReCaptchaEnterpriseProvider`).
- **Organization-Level Allowance:** Under Google Cloud Fraud Defense terms, reCAPTCHA Enterprise provides a free tier of **10,000 assessments per month** calculated at the **Google Cloud Organization / Billing Account level** (shared across all projects in that organization, not per project or app).
- **Distinction from Legacy Providers:** Conflating reCAPTCHA v3 with reCAPTCHA Enterprise is avoided. Standard reCAPTCHA v3 operates under legacy terms; reCAPTCHA Enterprise is the modern cloud-native provider integrated with Firebase App Check.

### 6.2 Token Acquisition, Token Refresh & Firestore Requests
The architecture strictly distinguishes three independent operations:
1. **Token Acquisition:** An App Check assessment occurs **only** when acquiring an initial token via `getToken()` on page load or user interaction.
2. **Token Refresh:** Handled automatically in the background by the Firebase SDK. To ensure seamless operation without unexpected session expiry, the client configuration **must explicitly set `isTokenAutoRefreshEnabled: true`**:
   ```javascript
   import { initializeAppCheck, ReCaptchaEnterpriseProvider } from 'firebase/app-check';

   const appCheck = initializeAppCheck(app, {
     provider: new ReCaptchaEnterpriseProvider(RECAPTCHA_ENTERPRISE_SITE_KEY),
     isTokenAutoRefreshEnabled: true // Explicitly configured; not assumed by default
   });
   ```
3. **Firestore Database Requests:** All database queries and writes attach the cached App Check token in the `X-Firebase-AppCheck` request header. The Firestore backend validates the token signature locally. **Firestore queries consume 0 additional reCAPTCHA assessments.**

### 6.3 Billing Configuration & The "Zero Surprise Bills" Condition
> [!IMPORTANT]
> **Strict Billing Condition:** The guarantee of "Zero surprise bills" holds **strictly and only when NO Google Cloud / Firebase billing account (Blaze plan) is attached to the project.**
- Under the no-cost **Spark Plan** (without billing attached), exhausting the 10,000 monthly assessments causes reCAPTCHA Enterprise to return HTTP 429 (`RESOURCE_EXHAUSTED`). Token acquisition fails and subsequent unverified Firestore writes are rejected, but zero financial charges are incurred.
- If a Google Cloud billing account is linked to the project, reCAPTCHA Enterprise automatically bills $1.00 per 1,000 assessments beyond the 10,000 monthly free tier. GitStore enforces the strict Spark configuration to eliminate financial liability.

### 6.4 Monthly Assessment Consumption (Architectural Estimates)
- **Storefront Browsing (`index.html`, `explore.html`, `product.html`):** 100% decoupled from Firebase. Anonymous visitors fetch pre-compiled static files directly from GitHub Pages. **Consumes 0 App Check assessments and 0 Firestore reads.**
- **Authenticated Developer Routes (`submit.html`, `dashboard.html`, `admin.html`):** App Check is lazy-loaded strictly on these pages.
- **Architectural Usage Projection (Unmeasured Engineering Estimate):**
  - Assumes 10–50 submission and claim operations per month.
  - Assumes 5–10 active developer visits per day (150–300 sessions/month).
  - Assumes average session duration of 30 minutes with token caching (1 token acquisition per session).
  - Projected Monthly Assessments: $\approx 200 - 1,000$ assessments/month (2% to 10% of the 10,000 free monthly quota).
  - *All usage projections are engineering estimates based on anticipated initial activity, not measured production telemetry.*

---

## 7. GitHub Token Strategy & Integration Test Classification

### 7.1 Required Permissions Matrix
To execute batch processing and collaborator verification with least privilege:
- `contents: write` (commit and push compiled static catalog and manifest)
- `pages: write` (upload Pages deployment artifact and dispatch deployment)
- `id-token: write` (OpenID Connect token minting for GitHub Pages deployment)
- `metadata: read` (read repository metadata and collaborator permissions)

### 7.2 Dual-Token Fallback Strategy
While GitHub documentation specifies that `Metadata: read` allows querying collaborator permissions, collaborator endpoints on personal repositories can return `403 Resource not accessible by integration` when using the default Actions `GITHUB_TOKEN`.
- **Primary:** Workflow attempts `GITHUB_TOKEN` with explicit permissions:
  ```yaml
  permissions:
    contents: write
    pages: write
    id-token: write
    metadata: read
  ```
- **Fallback:** If `GITHUB_TOKEN` receives HTTP 403 on `/collaborators/{username}/permission`, the worker falls back to `secrets.ADMIN_VERIFY_TOKEN` (a fine-grained Personal Access Token with repository `Metadata: read` permission).
- **Secret Handling & Rotation:** `ADMIN_VERIFY_TOKEN` is stored in GitHub Repository Secrets, restricted to default-branch runs, and rotated on a 90-day schedule.
- **Status Classification:** The collaborator permissions check and token strategy are formally classified as **PENDING INTEGRATION TEST**.

---

## 8. Software Safety Disclaimer & Legal Non-Execution

GitStore operates under an absolute non-execution invariant:
> "GitStore provides discovery and metadata verification only. GitStore does NOT execute, sandbox, or certify compiled binaries. Downloads are served directly from the author's public GitHub repository. Verify software source and run appropriate local anti-malware scanners prior to running executables."
