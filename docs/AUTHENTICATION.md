# GitStore — Authentication Architecture & Identity Specification

**Document Version:** 1.1.0  
**Phase:** Phase 0.1 — Architecture Audit, Hardening & Final Validation  
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

### 1.2 Policy Analysis: Prohibitions vs. Ambiguities

| Dimension | Explicitly Prohibited by GitHub | GitStore Design Approach | Status & Risk Assessment |
|---|---|---|---|
| **Password Transmission** | Sending passwords on GitHub Pages | **Strictly excluded.** No `<input type="password">` or password endpoints exist. | **Resolved.** 100% compliant. |
| **Credit Card / Payments** | Credit card numbers & commercial transactions | **Strictly excluded.** GitStore has no carts, fees, or payment processing. | **Resolved.** 100% compliant. |
| **Commercial SaaS** | Running a commercial SaaS web application | GitStore is a static directory for open software, not a paid SaaS. | **Resolved.** |
| **Third-Party Directory Submissions** | Not explicitly addressed in GitHub documentation | GitStore accepts crowdsourced metadata for open-source repositories. | **Ambiguity Remains:** Could automated platform scanners flag "marketplace" or submission activity? |
| **Monetization & Crowdfunding** | Explicitly permitted (donation & crowdfunding links allowed) | GitStore allows developer repository donation links (GitHub Sponsors). | **Permitted.** |

### 1.3 Mandatory Pre-Launch Policy Gate
> [!IMPORTANT]
> **Policy Compliance Launch Gate:** While eliminating password fields resolves the sensitive transaction rule, the architectural classification of a "software marketplace/directory" hosted on Pages remains subject to interpretation. GitStore **will not mark policy compliance as conclusively resolved** without formal written clarification. Obtaining written confirmation from GitHub Support is established as a formal **Phase 4 Pre-Launch Gate**.

---

## 2. Authentication Methods & User Flows

### 2.1 Primary Identity: GitHub OAuth (Popup Flow)
GitHub OAuth is the primary authentication vehicle for GitStore creators:
1. User clicks **[ CONNECT GITHUB ]** on `login.html` or the navbar.
2. The browser invokes `signInWithPopup(auth, new GithubAuthProvider())`.
3. Firebase opens a popup to `https://<project-id>.firebaseapp.com/__/auth/handler`, which redirects to GitHub’s official OAuth challenge:
   `https://github.com/login/oauth/authorize?client_id=...&scope=read:user,user:email`
4. The user completes authentication and 2FA directly on `github.com`. **No credentials touch GitHub Pages.**
5. GitHub issues an authorization code to the Firebase handler, which mints the Firebase session JWT and returns it to the client via cross-window messaging (`postMessage`).

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

## 3. Detailed Environment & Domain Configuration

### 3.1 Firebase Authorized Domains
To prevent unauthorized domains from invoking Firebase Auth, configure in Firebase Console (`Authentication -> Settings -> Authorized domains`):
- `localhost` (Local development)
- `127.0.0.1` (Local development fallback)
- `<project-id>.firebaseapp.com` (Default OAuth handler)
- `<project-id>.web.app` (Firebase project alias)
- `<username>.github.io` (Production GitHub Pages domain)

### 3.2 GitHub OAuth App Settings (on GitHub)
In GitHub (`Settings -> Developer settings -> OAuth Apps -> GitStore`):
- **Application Name:** `GitStore`
- **Homepage URL:** `https://<username>.github.io/GitStore/`
- **Authorization Callback URL:** `https://<project-id>.firebaseapp.com/__/auth/handler`

### 3.3 Email Continuation ActionCodeSettings
```javascript
export const getActionCodeSettings = () => {
  const isLocal = window.location.hostname === 'localhost' || window.location.hostname === '127.0.0.1';
  const baseUrl = isLocal 
    ? `${window.location.origin}/login.html`
    : `https://${window.location.hostname}/GitStore/login.html`;

  return {
    url: `${baseUrl}?finishSignUp=true`,
    handleCodeInApp: true,
  };
};
```

---

## 4. Multi-Provider Account Linking & Recovery

If a developer first signs in via Email Magic Link (`dev@example.com`) and later clicks "Sign in with GitHub" with an account verified under the same email:
1. Firebase Auth throws error: `auth/account-exists-with-different-credential`.
2. The UI intercepts the error, extracts `error.credential`, and prompts the user to sign in with their existing Email Link.
3. Once authenticated with Email Link, the client calls:
   ```javascript
   await linkWithCredential(auth.currentUser, pendingGithubCredential);
   ```
4. Both providers are now linked to the same primary Firebase UID, preserving the developer’s submission history.

---

## 5. Session Management & Persistence

- **Persistence Mode:** Local persistence (`browserLocalPersistence`) using sandboxed browser `IndexedDB`.
- **Logout:** Explicitly invokes `signOut(auth)`.
- **Auth Observer Pattern:**
  ```javascript
  import { onAuthStateChanged, setPersistence, browserLocalPersistence } from "firebase/auth";

  await setPersistence(auth, browserLocalPersistence);

  onAuthStateChanged(auth, async (user) => {
    if (user) {
      // Sync or fetch profile from Firestore /users/{uid}
      window.dispatchEvent(new CustomEvent('gitstore:auth_changed', { detail: { user } }));
    } else {
      window.dispatchEvent(new CustomEvent('gitstore:auth_changed', { detail: { user: null } }));
    }
  });
  ```

---

## 6. Firebase App Check Architecture & Realistic Limits

### 6.1 Provider Selection: reCAPTCHA v3
- **Selected Provider:** **reCAPTCHA v3** (Web).
- **Justification over reCAPTCHA Enterprise:** reCAPTCHA Enterprise charges $1.00 per 1,000 assessments after the first 10,000/month. On the Firebase Spark free plan without a billing account, exceeding 10,000 calls causes requests to fail. Standard reCAPTCHA v3 includes up to **1,000,000 calls per month at $0 cost**, ensuring strict adherence to zero-cost constraints.

### 6.2 Client Initialization & Local Debug Provider
```javascript
import { initializeAppCheck, ReCaptchaV3Provider, CustomProvider } from "firebase/app-check";

if (location.hostname === "localhost" || location.hostname === "127.0.0.1") {
  // Use debug token in local development
  self.FIREBASE_APPCHECK_DEBUG_TOKEN = true;
} else {
  initializeAppCheck(app, {
    provider: new ReCaptchaV3Provider('RECAPTCHA_V3_SITE_KEY'),
    isTokenAutoRefreshEnabled: true
  });
}
```

### 6.3 Crucial Architectural Limitations of App Check
> [!WARNING]
> **App Check is NOT Complete Security:**
> 1. **No Protection Against Authenticated Abusers:** App Check attests that requests come from a legitimate browser loading the GitStore web page. It does **not** restrict what an authenticated developer can do if they open DevTools and call the Firestore SDK directly.
> 2. **Does Not Replace Security Rules:** Security Rules remain the primary authorization boundary.
> 3. **Cannot Run on Actions Worker:** The GitHub Actions batch processor runs outside browser contexts and authenticates via `firebase-admin` with a private service account key, bypassing App Check completely.
