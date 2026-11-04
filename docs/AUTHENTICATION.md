# GitStore — Authentication Architecture & Identity Specification

**Document Version:** 1.9.0  
**Phase:** Phase 0.1.8 — Node 24 Workflow Compatibility, Artifact Provenance & Transactional Deployment Integrity  
**Status:** IN_REVIEW — Awaiting Explicit User Approval  
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
6. Client extracts this verified ID using provider-specific search (never assuming it is the first entry) and normalizes it:
   ```javascript
   function normalizeGithubId(val) {
     if (val == null) return null;
     const str = String(val).trim();
     if (!/^\d+$/.test(str)) return null;
     return str;
   }

   const githubProvider = auth.currentUser.providerData.find(
     p => p.providerId === 'github.com'
   );
   const verifiedGithubNumericId = normalizeGithubId(githubProvider?.uid); // Immutable numeric string e.g. "583231"
   ```

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
3. Worker verifies ownership matching author GitHub numeric ID and updates submission atomically.

---

## 4. Multi-Provider Account Linking & Mixed-Provider Analysis

### 4.1 Multi-Provider Account Linking
If a developer first signs in via Email Magic Link (`dev@example.com`) and later wants to link their GitHub account:
1. The developer invokes `linkWithPopup(auth.currentUser, new GithubAuthProvider())`.
2. Firebase Auth links the `github.com` provider to the existing Firebase user record.
3. Once linked, `userRecord.providerData` contains the `github.com` entry, allowing the developer to claim their Issue Form submissions seamlessly.

### 4.2 Mixed-Provider Step-Up Reauthentication Empirical Test
- **9-Step Test Procedure:**
  1. Create test user via `sendSignInLinkToEmail()`.
  2. Complete email link authentication via `signInWithEmailLink()`.
  3. Call `linkWithPopup(auth.currentUser, new GithubAuthProvider())` to attach GitHub credentials.
  4. Log user out completely (`signOut()`).
  5. Log back in strictly using email link (`signInWithEmailLink()`).
  6. Inspect baseline token: `const tokenA = await auth.currentUser.getIdTokenResult();` (Assert `tokenA.claims.firebase.sign_in_provider == 'emailLink'`).
  7. Call interactive step-up reauthentication: `await reauthenticateWithPopup(auth.currentUser, new GithubAuthProvider());`.
  8. Fetch refreshed token: `const tokenB = await auth.currentUser.getIdTokenResult(true);`.
  9. Assert whether `tokenB.claims.firebase.sign_in_provider` switched to `'github.com'` or remained `'emailLink'`.
- **Classification:** Classified as **`PENDING_INTEGRATION_TEST`**.
- **Operational Requirement:** Administrators executing moderation commands must authenticate their active session directly via GitHub OAuth to satisfy the `sign_in_provider == 'github.com'` security rule gate.

---

## 5. Firebase App Check: Provider, Quotas & Caching Architecture

### 5.1 Provider Selection: reCAPTCHA Enterprise
- **Recommended Provider:** **reCAPTCHA Enterprise** (`ReCaptchaEnterpriseProvider`).
- **Organization-Level Allowance:** Google Cloud provides a no-cost tier of **10,000 assessments per month** for reCAPTCHA Enterprise under Fraud Defense, calculated across the Google Cloud Organization.
- Configured with `isTokenAutoRefreshEnabled: true`.
- App Check is lazy-loaded **only on pages that interact with Cloud Firestore**: `submit.html`, `dashboard.html`, `admin.html`. Storefront visitors load static files from GitHub Pages with **0 App Check assessments**.

---

## 6. Administrator Bootstrap & Identity Verification

### 6.1 Root Administrator Verification Flow
Repository ownership of `Rohitpvt/GitStore` alone does not authenticate the person making an administrative request:
1. Retrieve canonical repository owner's immutable numeric GitHub account ID from trusted GitHub repository metadata (`GET /repos/Rohitpvt/GitStore` -> `response.data.owner.id`).
2. Verify caller's Firebase ID token: `admin.auth().verifyIdToken(idToken, true)`.
3. Locate `userRecord.providerData.find(p => p.providerId === 'github.com')`. If absent, reject with `403 FORBIDDEN`.
4. Normalize and compare IDs: `normalizeGithubId(githubProvider.uid) === normalizeGithubId(repoMetadata.owner.id)`. If true, caller is authenticated as root administrator.

---

## 7. Action-Bound Step-Up Authentication Proof Integrity & Replay Protection (Option A)

To eliminate the risks of stolen session tokens, cross-tab session hijacking, or unauthorized administrative actions performed through linked email accounts, GitStore enforces **Mandatory Fresh GitHub Step-Up Reauthentication with Action-Bound Proofs and Atomic Nonce Consumption (Option A)** for all privileged operations (`APPROVE_SUBMISSION`, `REJECT_SUBMISSION`, `VERIFY_DEVELOPER`, `REVOKE_DEVELOPER`, `DELIST_PRODUCT`, `FEATURE_PRODUCT`):

### 7.1 Interactive Step-Up Challenge Flow
1. When an administrator initiates a moderation action on `admin.html`, the client triggers:
   ```javascript
   import { getAuth, GithubAuthProvider, reauthenticateWithPopup } from 'firebase/auth';

   const auth = getAuth();
   const provider = new GithubAuthProvider();
   provider.addScope('read:user');

   // Forces interactive GitHub authentication popup
   const result = await reauthenticateWithPopup(auth.currentUser, provider);
   const tokenResult = await auth.currentUser.getIdTokenResult(true);
   ```

2. **ActionHash Classification & Cryptographic Reality:**
   **No raw OAuth access tokens are stored in Cloud Firestore.**
   Instead, the client constructs an action-bound proof:
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
   > **Integrity Checksum Classification:** The client-generated SHA-256 `actionHash` contains no server secret or private cryptographic key. It is **not** an independent digital signature or authentication proof. It functions strictly as a **client data-integrity checksum** ensuring that the tuple `(action, targetDocumentPath, actionNonce, authTime)` was not corrupted or modified in transit.

3. **Atomic Nonce Reservation in `/consumed_stepup_nonces`:**
   To guarantee that an action nonce cannot be reused across different request documents:
   - When the worker leases a moderation request, it calculates `nonceHash = SHA256(actionNonce)`.
   - In the exact same atomic Firestore transaction that transitions the request from `pending` to `processing`, the worker asserts that `/consumed_stepup_nonces/{nonceHash}` does not exist.
   - If it exists, the transaction fails and the request is marked `rejected` (`NONCE_ALREADY_CONSUMED`).
   - If it does not exist, the worker creates `/consumed_stepup_nonces/{nonceHash}` with `{ nonceHash, requestId, requestedByUid: req.requestedByUid, action: req.action, targetDocumentPath: req.targetDocumentPath, workerRunId, consumedAt }`.
   - **Recovery Policy:** If the worker terminates unexpectedly post-consumption (e.g. runner SIGKILL or GitHub API 500), the nonce remains consumed. The administrator must initiate a new action in the dashboard, generating a fresh nonce.
   - Client access to `/consumed_stepup_nonces` is default-deny (`allow read, write: if false;`).

4. **Action-Specific Moderation Payload Schemas & Target Path Validation:**
   The worker and Security Rules validate that `targetDocumentPath` matches canonical targets derived from `action` and `targetId`. Furthermore, `payload` is validated against action-specific nested schemas (Section 3.2 of [`docs/DATABASE_SCHEMA.md`](./DATABASE_SCHEMA.md)), rejecting arbitrary nested keys or unapproved properties.

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
       && (
         (request.resource.data.action in ['APPROVE_SUBMISSION', 'REJECT_SUBMISSION'] 
           && request.resource.data.targetId.matches('^[a-zA-Z0-9_-]+$')
           && request.resource.data.targetDocumentPath == 'submissions/' + request.resource.data.targetId)
         || (request.resource.data.action in ['VERIFY_DEVELOPER', 'REVOKE_DEVELOPER']
           && request.resource.data.targetId.matches('^[a-zA-Z0-9_-]+$')
           && (request.resource.data.targetDocumentPath == 'users/' + request.resource.data.targetId || request.resource.data.targetDocumentPath == 'public_profiles/' + request.resource.data.targetId))
         || (request.resource.data.action in ['DELIST_PRODUCT', 'FEATURE_PRODUCT']
           && request.resource.data.targetId.matches('^prod_[0-9]+_[a-z0-9-]+$')
           && request.resource.data.targetDocumentPath == 'products/' + request.resource.data.targetId)
       )
       && validModerationPayload(request.resource.data.action, request.resource.data.payload)
       && request.resource.data.stepUpProof.keys().hasOnly([
         'providerId', 'authTime', 'actionNonce', 'targetDocumentPath', 'actionHash'
       ])
       && request.resource.data.stepUpProof.providerId == 'github.com'
       && request.resource.data.stepUpProof.authTime == request.auth.token.auth_time
       && request.resource.data.stepUpProof.targetDocumentPath == request.resource.data.targetDocumentPath
       && request.resource.data.stepUpProof.actionHash.size() == 64
       && request.resource.data.status == 'pending'
       && request.resource.data.createdAt == request.time;
     allow update, delete: if false; // Immutability prevents tampering
   }
   ```
