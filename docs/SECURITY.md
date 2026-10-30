# GitStore — Security Architecture & Threat Model

**Document Version:** 1.2.0  
**Phase:** Phase 0.1.1 — Critical Architecture Corrections & Hardening  
**Status:** Approved Security Specification (Corrected)  

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
| **Claim Processing Lease** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Transactional superuser claim |
| **Verify Repository & Asset**| ❌ Forbidden | ❌ Client assertion only | ❌ Client assertion only | ✅ Authoritative GitHub REST API checks |
| **Approve / Publish Product** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues via `moderation_requests` | ✅ Authoritative state transition |
| **Grant Developer Verification** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues via `moderation_requests` | ✅ Sets `isVerifiedDeveloper: true` |
| **Write to `admin_logs`** | ❌ Forbidden | ❌ Forbidden | ❌ Forbidden | ✅ Immutable append-only write |
| **Delist Live Product** | ❌ Forbidden | ❌ Forbidden | ❌ Enqueues via `moderation_requests` | ✅ Transitions product to `delisted` |

---

## 3. Audited Administrator Security Workflow (Trust Boundary Corrected)

### 3.1 The Canonical GitStore Repository Boundary
The critical trust boundary flaw in baseline specifications was checking collaborator status against the *submitted* application's repository. In GitStore v1.2.0:
- Administrator permissions are verified **strictly against GitStore's own canonical repository** (`Rohitpvt/GitStore`), injected via `process.env.GITHUB_REPOSITORY`.

### 3.2 Cryptographic Identity Extraction (Zero Browser Trust)
Browser-submitted usernames are never trusted:
1. The admin authenticates in the browser via GitHub OAuth and writes to `/moderation_requests/{requestId}` with `requestedByUid: auth.uid`.
2. The GitHub Actions worker queries the Firebase Admin SDK:
   ```javascript
   const userRecord = await admin.auth().getUser(request.requestedByUid);
   const ghProvider = userRecord.providerData.find(p => p.providerId === 'github.com');
   if (!ghProvider) throw new Error("Caller has no verified GitHub identity");
   const verifiedNumericId = ghProvider.uid;
   ```
3. The worker calls GitHub API `GET /user/{verifiedNumericId}` to resolve the immutable GitHub login.
4. The worker checks collaborator rights on GitStore's repository:
   `GET /repos/{CANONICAL_OWNER}/{CANONICAL_REPO}/collaborators/{verifiedLogin}/permission`
   Must return `permission: "admin"` or `"maintain"`.
5. If the user was removed as a collaborator on GitHub, the API immediately returns 404 or `read`, and the request is rejected with an audit log. Stale Firestore claims are impossible.

### 3.3 Initial Administrator Bootstrap
- The repository owner (`Rohitpvt`, derived from `process.env.GITHUB_REPOSITORY_OWNER`) is automatically the root administrator.
- New maintainers are established simply by adding them as repository collaborators on GitHub. Zero client-side role elevation exists.

---

## 4. Repository Ownership Verification & Anti-Hijacking

### 4.1 Elimination of Placeholder Accounts
- Issue Form submissions do **not** create fake Firebase Auth accounts.
- Submissions are stored with `submitterGithubId` and `claimStatus: 'unclaimed'`.
- When the developer logs into GitStore via GitHub OAuth, their authentic Firebase Auth account claims the submission seamlessly.

### 4.2 Multi-Tier Ownership Verification Matrix

| Tier | Submitter Status | Evidence Checked via GitHub API | What it Proves | Eligible for Auto-Publish? | Action Taken |
|---|---|---|---|---|---|
| **Tier 1: Direct Personal Owner** | Personal repo owner | `repo.owner.id == submitter.githubId` | Complete administrative control of repository | **YES** (if developer is verified) | Auto-publishes on valid asset checks |
| **Tier 2: Organization Member** | Org repo submitter | `GET /orgs/{org}/public_members/{login}` (204) | Public affiliation with org | **NO** (does not prove repo release authority) | Gated to `needs_review` |
| **Tier 3: Committer** | Contributor to repo | `GET /repos/{owner}/{repo}/commits?author={login}` | Has authored merged commits | **NO** (does not prove release authorization) | Gated to `needs_review` |
| **Tier 4: Community Nomination**| Fan / Third-party | No ownership or commit proof found | Public open-source suggestion only | **NO** (unverified submitter) | Gated to `needs_review` |

---

## 5. Firebase App Check Architecture & Assessment Quotas

1. **Provider:** **reCAPTCHA v3** (Web).
2. **Quota Reality:** Under Google Cloud / Firebase terms, the no-cost allowance is **10,000 assessments per month**.
3. **Lazy-Loading Mitigation:**
   - Storefront visitors browsing `index.html`, `explore.html`, and `product.html` read static JSON directly from GitHub Pages and **do not initialize App Check**.
   - App Check is initialized **only on authenticated developer pages** (`submit.html`, `dashboard.html`, `admin.html`).
   - 50 active developers/day spending 30 minutes generate ~1,500–3,000 assessments/month, staying safely within the 10,000 free assessment ceiling.
4. **App Check Limitations:**
   > [!WARNING]
   > App Check validates browser origin, not user intent. Firestore Security Rules remain the primary line of defense.

---

## 6. Software Safety Disclaimer & Legal Non-Execution

GitStore operates under an absolute non-execution invariant:
> "GitStore provides discovery and metadata verification only. GitStore does NOT execute, sandbox, or certify compiled binaries. Downloads are served directly from the author's public GitHub repository. Verify software source and run appropriate local anti-malware scanners prior to running executables."
