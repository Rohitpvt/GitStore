# GitStore — Authentication Architecture & Identity Specification

**Document Version:** 1.2.0  
**Phase:** Phase 0.1.1 — Critical Architecture Corrections & Hardening  
**Status:** Approved Authentication Specification (Corrected)  
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
5. GitHub issues an authorization code to the Firebase handler, which mints the Firebase session JWT containing the verified, immutable numeric GitHub user ID in `user.providerData[0].uid`.
6. Client extracts this verified ID. Browser-submitted text is **never** used as identity proof.

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
- Submissions are stored with `submitterGithubId: issue.user.id` and `claimStatus: 'unclaimed'`.

### 3.2 Claiming Submissions upon OAuth Login
When a developer signs into GitStore via GitHub OAuth:
1. Firebase Auth returns their authenticated Firebase UID (`user.uid`) and verified numeric GitHub user ID (`user.providerData[0].uid`).
2. The client or Actions worker queries `/submissions` where:
   `submitterGithubId == user.providerData[0].uid && claimStatus == 'unclaimed'`
3. For each match, updates:
   ```typescript
   developerUid: user.uid,
   claimStatus: 'claimed'
   ```
4. This securely claims the submission without creating fake accounts or risking account takeover.

### 3.3 Multi-Provider Account Linking
If a developer first signs in via Email Magic Link (`dev@example.com`) and later clicks "Sign in with GitHub" with an account verified under the same email:
1. Firebase Auth throws error: `auth/account-exists-with-different-credential`.
2. The UI prompts the user to sign in with their existing Email Link.
3. Once authenticated with Email Link, the client calls:
   ```javascript
   await linkWithCredential(auth.currentUser, pendingGithubCredential);
   ```
4. Both providers link to the same primary Firebase UID.

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

## 5. Firebase App Check: Quota Reality & Lazy-Loading Strategy

### 5.1 Provider Selection & Quota Reality
- **Selected Provider:** **reCAPTCHA v3** (Web).
- **Official Google Cloud / Firebase Quota:** **10,000 assessments per month free**.
- **Assessment Consumption:** Each App Check token generation or auto-refresh triggers a backend assessment. With a default 1-hour TTL (auto-refreshed at 30 minutes), each active session consumes ~2 assessments per hour.

### 5.2 The Lazy-Loading Architecture
If App Check was initialized on public storefront browsing pages (`index.html`, `explore.html`), thousands of anonymous visitors would exhaust the 10,000 assessment free tier within days, causing App Check token generation to fail and blocking legitimate users.

**GitStore isolates App Check strictly to authenticated developer pages:**
- **Storefront Visitors:** Browse static catalogs (`data/catalog.json`) served directly by GitHub Pages. They make **0 Firestore requests** and **do NOT initialize App Check**.
- **Authenticated Developers:** App Check is lazy-loaded **only on pages that interact with Cloud Firestore**:
  - `submit.html` (Submissions wizard)
  - `dashboard.html` (Developer dashboard)
  - `admin.html` (Admin dashboard)
- **Math:** 50 active developers/day spending 30 minutes = ~50–100 assessments/day = ~1,500–3,000 assessments/month, safely below the 10,000 monthly free tier.

### 5.3 Phased Enforcement Strategy
- **Phase 2 (Integration):** App Check initialized in **metrics/monitoring mode** (tokens validated, but unverified requests are not dropped).
- **Phase 6 (Pre-Launch QA):** App Check switched to **strict enforcement** only after verifying zero false-positive dropouts.
