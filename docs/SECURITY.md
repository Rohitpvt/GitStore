# GitStore — Security Architecture & Threat Model

**Document Version:** 1.3.0  
**Phase:** Phase 0.1.2 — Final Security & Architecture Corrections  
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

## 3. Audited Administrator Security Workflow (Trust Boundary Corrected)

### 3.1 The Canonical GitStore Repository Boundary Check
Administrator authorization is verified strictly against **GitStore's own canonical repository** (`Rohitpvt/GitStore`).
To prevent privilege escalation from forks, the worker explicitly validates its execution context:
```javascript
if (process.env.GITHUB_REPOSITORY !== 'Rohitpvt/GitStore') {
  throw new Error(`SECURITY ALERT: Execution on non-canonical repository '${process.env.GITHUB_REPOSITORY}' rejected.`);
}
```
If a fork runs this workflow, it aborts immediately before accessing any production data or secrets.

### 3.2 Cryptographic Identity Extraction (Zero Browser Trust)
Browser-submitted usernames and claims are completely ignored:
1. The admin authenticates in the browser via GitHub OAuth and writes to `/moderation_requests/{requestId}` with `requestedByUid: request.auth.uid`.
2. The GitHub Actions worker queries the Firebase Admin SDK:
   ```javascript
   const userRecord = await admin.auth().getUser(request.requestedByUid);
   const ghProvider = userRecord.providerData.find(p => p.providerId === 'github.com');
   if (!ghProvider || !ghProvider.uid) {
     throw new Error("UNAUTHORIZED: Caller has no verified GitHub identity linked in Firebase Auth");
   }
   const verifiedNumericId = ghProvider.uid;
   ```
3. The worker calls GitHub API `GET https://api.github.com/user/{verifiedNumericId}` to resolve the current GitHub login handle from the immutable numeric ID.

### 3.3 Fail-Closed Collaborator Permission Check
The worker queries GitHub's official endpoint:
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

**Inherited Permissions & Team Grants:**
GitHub's endpoint calculates the highest effective permission and returns it in `role_name`, whether granted directly, through team memberships, or via organization-level owner/maintainer privileges.

**Token Permissions & Integration Status:**
Calling `/repos/{owner}/{repo}/collaborators/{username}/permission` requires `Metadata: read` for fine-grained PAT or GitHub App tokens. However, the default GitHub Actions `GITHUB_TOKEN` frequently receives `403 Resource not accessible by integration` on collaborator management endpoints due to repository-level scope limits. The worker implements a dual-token strategy:
- Primary: attempts `secrets.GITHUB_TOKEN`.
- Fallback: uses `secrets.ADMIN_VERIFY_TOKEN` (a dedicated fine-grained PAT or GitHub App token with `Metadata: read` stored in repository secrets).
*Classification: Pending live integration verification.*

### 3.4 Audit Logging Schema
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

### 3.5 Initial Administrator Bootstrap
- The repository owner (`Rohitpvt`, derived from canonical repository configuration) is inherently the root administrator.
- Additional maintainers are established simply by adding them as repository collaborators on `Rohitpvt/GitStore` in GitHub repository settings. Zero client-side role elevation exists, preventing any bootstrap backdoor.

---

## 4. Repository Ownership, Release Authority & Verified Developer

### 4.1 Distinction of Three Security Concepts
1. **Repository Ownership:** Proves control over the GitHub repository entity (`repo.owner.id === submitterGithubId`).
2. **Release Authority:** Proves authorization to publish executable software release binaries. Direct personal owners possess inherent release authority. Organization members, committers, and third parties do **not** possess proven release authority and are held in `needs_review`.
3. **Verified Developer Identity:** A platform trust badge awarded by GitStore administrators based on formal criteria (e.g. established open-source track record, GPG-signed commits, or manual review).
> [!IMPORTANT]
> A repository owner does **NOT** automatically receive a "Verified Developer" badge merely because they own the repository. Owning a personal repo permits auto-publication of that repo's valid releases, but does not confer platform-wide verified status.

### 4.2 Multi-Tier Ownership Verification Matrix

| Tier | Submitter Status | Evidence Checked via GitHub API | What it Proves | Eligible for Auto-Publish? | Action Taken |
|---|---|---|---|---|---|
| **Tier 1: Direct Personal Owner** | Personal repo owner | `repo.owner.id == submitter.githubId` | Complete administrative control of repository | **YES** (if valid assets & release) | Auto-publishes on valid asset checks |
| **Tier 2: Organization Member** | Org repo submitter | `GET /orgs/{org}/public_members/{login}` (204) | Public affiliation with org | **NO** (does not prove repo release authority) | Gated to `needs_review` |
| **Tier 3: Committer** | Contributor to repo | `GET /repos/{owner}/{repo}/commits?author={login}` | Has authored merged commits | **NO** (does not prove release authorization) | Gated to `needs_review` |
| **Tier 4: Community Nomination**| Fan / Third-party | No ownership or commit proof found | Public open-source suggestion only | **NO** (unverified submitter) | Gated to `needs_review` |

### 4.3 Secure Claiming Workflow (`/claim_requests`)
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
4. **Firestore Security Rules Protection:**
   - `allow create: if request.auth != null && request.resource.data.requesterUid == request.auth.uid && request.resource.data.status == 'pending' && request.resource.data.createdAt == request.time;`
   - `allow update, delete: if false;` (Clients can NEVER alter or delete claim requests).
   - In `/submissions/{id}`, browser clients have `allow update: if false;` for fields `developerUid`, `claimStatus`, `ownershipProof`.
5. The trusted worker validates caller's linked `github.com` provider ID via Firebase Admin SDK. If `provider.uid === targetSubmission.submitterGithubId`, worker executes an atomic Firestore transaction assigning ownership and setting `claimStatus: 'claimed'`.
6. **Separate Email & GitHub Accounts:** If a developer signs in via Email Magic Link, they must call `linkWithPopup(auth.currentUser, new GithubAuthProvider())` to link their GitHub account. Once linked, `providerData` contains the `github.com` provider, enabling claim validation.

---

## 5. Firebase App Check Provider & Quota Reality

### 5.1 Provider Choice: reCAPTCHA Enterprise
- **Recommended Provider:** **reCAPTCHA Enterprise** (`ReCaptchaEnterpriseProvider`).
- **Source-Backed Allowance:** Under Google Cloud / Firebase terms, reCAPTCHA Enterprise includes a no-cost tier of **10,000 assessments per month**.
- **Provider Distinction:** Conflating reCAPTCHA v3 with reCAPTCHA Enterprise is avoided. Standard reCAPTCHA v3 is part of legacy reCAPTCHA admin console terms; reCAPTCHA Enterprise is the modern provider managed under Google Cloud Fraud Defense.

### 5.2 Token TTL, Caching & Assessment Consumption
- **Assessment vs. Firestore Request:** An App Check assessment occurs **only** when acquiring or refreshing an App Check token (`assessment.create()`), not on every individual Firestore document read or write.
- **Token TTL:** App Check tokens have a default TTL of **1 hour** and are auto-refreshed by the Firebase Web SDK at approximately 30–45 minutes.
- **Token Caching:** All Firestore operations during an active session attach the cached App Check token in HTTP headers (`X-Firebase-AppCheck`). The Firestore backend verifies the token signature locally with zero extra reCAPTCHA assessment calls.

### 5.3 Realistic Monthly Assessment Consumption (Architectural Estimates)
- **Storefront Browsing (`index.html`, `explore.html`, `product.html`):** 100% decoupled from Firebase. Anonymous visitors fetch static files directly from GitHub Pages. **Consumes 0 App Check assessments.**
- **Authenticated Developer Routes (`submit.html`, `dashboard.html`, `admin.html`):** App Check is lazy-loaded strictly on these pages.
- **Consumption Estimate:**
  - 50 active developers/day $\times$ 4 visits/month = 200 sessions/month.
  - Average session duration: 30 minutes $\rightarrow$ 1 assessment per session.
  - Estimated Monthly Assessments: $\approx 200 - 1,000$ assessments/month (2% to 10% of the 10,000 free monthly quota).
- **Quota Exceeded Behavior:** If the 10,000 monthly allowance is exceeded on the Spark plan without a billing account, reCAPTCHA Enterprise returns HTTP 429 (`RESOURCE_EXHAUSTED`). Token acquisition fails, causing backend App Check enforcement to reject subsequent Firestore writes.

---

## 6. Software Safety Disclaimer & Legal Non-Execution

GitStore operates under an absolute non-execution invariant:
> "GitStore provides discovery and metadata verification only. GitStore does NOT execute, sandbox, or certify compiled binaries. Downloads are served directly from the author's public GitHub repository. Verify software source and run appropriate local anti-malware scanners prior to running executables."
