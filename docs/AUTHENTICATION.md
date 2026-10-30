# GitStore — Authentication Architecture & Identity Specification

**Document Version:** 1.3.0  
**Phase:** Phase 0.1.2 — Final Security & Architecture Corrections  
**Status:** Approved Authentication Specification (Hardened)  
**Provider:** Firebase Authentication (v10 Modular SDK)  
**Host Environment:** GitHub Pages (`https://<org>.github.io/<repo>/`)

---

## 1. Compliance with GitHub Pages Policies & Launch Gate

### 1.1 Official Policy Citations
GitHub’s official published documentation regarding GitHub Pages establishes explicit limits:

1. **Sensitive Transactions:**
   > *"GitHub Pages sites shouldn't be used for sensitive transactions like sending passwords or credit card numbers."*  
   > — Source: [GitHub Docs: About GitHub Pages / Limits on use of GitHub Pages](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages#limits-on-use-of-github-pages)

2. **Commercial & Online Business Restrictions:**
   > *"GitHub Pages is not intended for or allowed to be used as a free web-hosting service to run your online business, e-commerce site, or any other website that is primarily directed at either facilitating commercial transactions or providing commercial software as a service (SaaS)."*  
   > — Source: [GitHub Terms: Additional Products and Features](https://docs.github.com/en/site-policy/github-terms/github-terms-of-service)

### 1.2 Mandatory Pre-Launch Policy Gate
> [!IMPORTANT]
> **Pre-Launch Policy Gate:** While eliminating password fields resolves the sensitive transaction rule, the architectural classification of an open software directory hosted on Pages remains subject to interpretation. GitStore **will not mark policy compliance as conclusively resolved** without formal written clarification. Obtaining written confirmation from GitHub Support is established as a formal **Phase 4 Pre-Launch Gate**.

---

## 2. Authentication Methods & User Flows

### 2.1 Primary Identity: GitHub OAuth (Popup Flow)
GitHub OAuth is the primary authentication vehicle for GitStore creators:
1. User clicks **[ CONNECT GITHUB ]** on `login.html` or the navbar.
2. The browser invokes `signInWithPopup(auth, new GithubAuthProvider())`.
3. Firebase opens a popup to `https://<project-id>.firebaseapp.com/__/auth/handler`, which redirects to GitHub’s official OAuth challenge:
   `https://github.com/login/oauth/authorize?client_id=...&scope=read:user,user:email`
4. The user completes authentication directly on `github.com`. **No credentials touch GitHub Pages.**
5. GitHub issues an authorization code to the Firebase handler, which mints the Firebase session JWT containing the verified, immutable numeric GitHub user ID in `user.providerData`.
6. Client extracts this verified ID using provider-specific search (never assuming it is the first entry):
   ```javascript
   const githubProvider = auth.currentUser.providerData.find(
     p => p.providerId === 'github.com'
   );
   const verifiedGithubNumericId = githubProvider?.uid; // Immutable numeric string e.g. "583231"
   ```
   Browser-submitted text is **never** trusted as proof of identity.

### 2.2 Secondary Identity: Passwordless Email Magic Link
For developers without immediate GitHub access:
1. Developer submits their email address on `login.html`.
2. Client calls `sendSignInLinkToEmail(auth, email, actionCodeSettings)`.
3. Firebase dispatches an email containing a secure cryptographic sign-in URL with an embedded one-time token.
4. User clicks the email link, opening:
   `https://<username>.github.io/GitStore/login.html?finishSignUp=true&apiKey=...`
5. Client verifies `isSignInWithEmailLink(auth, window.location.href)` and calls `signInWithEmailLink(auth, email, window.location.href)`.
6. **Zero passwords are typed, captured, or transmitted.**

---

## 3. Account Linking & Claiming Issue Form Submissions

### 3.1 No Placeholder Firebase Accounts
- Submissions created via GitHub Issue Forms do **not** generate placeholder Firebase Auth accounts.
- Submissions are stored with `submitterGithubId: issue.user.id` (immutable numeric ID), `developerUid: null`, and `claimStatus: 'unclaimed'`.

### 3.2 Secure Claiming Workflow (`/claim_requests`)
Because Cloud Functions are excluded and browser clients must never directly update submission ownership fields, ownership claiming uses an asynchronous, worker-mediated command queue:
1. The developer authenticates on GitStore via Firebase Auth.
2. The developer initiates a claim from `dashboard.html`, writing to `/claim_requests/{requestId}`:
   ```typescript
   {
     requestId: string;
     requesterUid: string;              // Must equal request.auth.uid (enforced by Security Rules)
     targetSubmissionId: string;        // Foreign key to submissions/{id}
     status: 'pending';
     createdAt: serverTimestamp();
   }
   ```
3. **Firestore Security Rules Protection:**
   ```javascript
   match /claim_requests/{requestId} {
     allow read: if request.auth != null && resource.data.requesterUid == request.auth.uid;
     allow create: if request.auth != null 
       && request.resource.data.requesterUid == request.auth.uid
       && request.resource.data.status == 'pending'
       && request.resource.data.createdAt == request.time;
     allow update, delete: if false; // Worker only
   }
   ```
4. **Worker Execution Logic:**
   - The scheduled batch worker fetches `admin.auth().getUser(claimReq.requesterUid)`.
   - The worker locates `userRecord.providerData.find(p => p.providerId === 'github.com')`. If absent, the claim is rejected (`rejected_no_github_provider`).
   - The worker extracts `verifiedGithubId = githubProvider.uid`.
   - The worker reads target submission:
     - Verifies `submission.submitterGithubId === verifiedGithubId`.
     - Verifies `submission.claimStatus === 'unclaimed'`.
   - In a Firestore transaction:
     - Sets `submission.developerUid = claimReq.requesterUid`
     - Sets `submission.claimStatus = 'claimed'`
     - Sets `submission.claimedAt = serverTimestamp()`
     - Sets `claimReq.status = 'completed'`
     - Writes audit entry to `/admin_logs`
   - If IDs do not match, the worker sets `claimReq.status = 'rejected_mismatch'`. If already claimed, sets `rejected_already_claimed`.

### 3.3 Multi-Provider Account Linking
If a developer first signs in via Email Magic Link (`dev@example.com`) and later wants to link their GitHub account:
1. The developer invokes `linkWithPopup(auth.currentUser, new GithubAuthProvider())`.
2. Firebase Auth links the `github.com` provider to the existing Firebase user record.
3. Once linked, `userRecord.providerData` contains the `github.com` entry, allowing the developer to claim their Issue Form submissions seamlessly.

---

## 4. Detailed Environment & Domain Configuration

### 4.1 Firebase Authorized Domains
In Firebase Console (`Authentication -> Settings -> Authorized domains`), configure:
- `localhost`
- `127.0.0.1`
- `<project-id>.firebaseapp.com`
- `<project-id>.web.app`
- `<username>.github.io`

### 4.2 GitHub OAuth App Settings (on GitHub)
In GitHub (`Settings -> Developer settings -> OAuth Apps -> GitStore`):
- **Application Name:** `GitStore`
- **Homepage URL:** `https://<username>.github.io/GitStore/`
- **Authorization Callback URL:** `https://<project-id>.firebaseapp.com/__/auth/handler`

---

## 5. Firebase App Check: Provider, Quotas & Caching Architecture

### 5.1 Provider Selection: reCAPTCHA Enterprise
- **Recommended Provider:** **reCAPTCHA Enterprise** (`ReCaptchaEnterpriseProvider`).
- **Official Google Cloud Allowance:** Google Cloud provides a no-cost tier of **10,000 assessments per month** for reCAPTCHA Enterprise under Fraud Defense.
- **Provider Distinction:** Standard reCAPTCHA v3 is part of legacy reCAPTCHA admin terms; reCAPTCHA Enterprise is the modern provider managed under Google Cloud and recommended by Firebase for all new web integrations.

### 5.2 Token TTL, Caching & Request Separation
- **Assessments vs. Service Requests:** An App Check assessment occurs **only** when minting or refreshing an App Check token (`getToken()`), not on every Firestore document request.
- **Token TTL:** App Check tokens have a default TTL of **1 hour** and are auto-refreshed by the Firebase Web SDK at roughly 30–45 minutes.
- **Token Caching:** The Firebase SDK caches the token locally. When queries or writes are sent to Cloud Firestore, the cached token is attached in the `X-Firebase-AppCheck` header. The Firebase backend validates the signature locally without contacting reCAPTCHA.

### 5.3 The Lazy-Loading Architecture & Usage Estimates
- **Storefront Visitors:** Browse static catalogs (`data/catalog.json`) served directly by GitHub Pages. They make **0 Firestore requests** and **do NOT initialize App Check**.
- **Authenticated Developers:** App Check is lazy-loaded **only on pages that interact with Cloud Firestore**:
  - `submit.html` (Submissions wizard)
  - `dashboard.html` (Developer dashboard)
  - `admin.html` (Admin dashboard)
- **Architectural Consumption Estimate:**
  - 50 active developers/day $\times$ 4 visits/month = 200 sessions/month.
  - Average session: 30 minutes $\rightarrow$ 1 assessment per session.
  - Estimated Monthly Assessments: $\approx 200 - 1,000$ assessments/month (2% to 10% of the 10,000 free monthly quota).
  - *All estimates are architectural projections, not guaranteed free usage.*
- **Quota Exceeded Behavior:** If assessments exceed 10,000 in a month on the Spark plan without a billing account, reCAPTCHA Enterprise returns HTTP 429 (`RESOURCE_EXHAUSTED`). Token acquisition fails, causing backend App Check enforcement to reject subsequent Firestore writes.

### 5.4 Phased Enforcement Strategy
- **Phase 2 (Integration):** App Check initialized in **metrics/monitoring mode** (tokens validated, but unverified requests are not dropped).
- **Phase 6 (Pre-Launch QA):** App Check switched to **strict enforcement** only after verifying zero false-positive dropouts.
