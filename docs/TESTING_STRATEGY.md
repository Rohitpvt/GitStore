# GitStore — Comprehensive Testing Strategy

**Document Version:** 1.6.0  
**Phase:** Phase 0.1.5 — Final Security Rules, Administrator Step-Up Authentication & Deployment Recovery Audit  
**Status:** Approved Quality Assurance & Testing Framework (Hardened & Reconciled)  

---

## 1. Testing Philosophy & Quality Invariants

GitStore adheres to a layered testing hierarchy designed to guarantee platform reliability, client security, and pipeline idempotency without incurring external testing costs.

### Testing Invariants
1. **Zero-Trust Security Verification:** All Firestore Security Rules must be proven secure using automated unit tests executed against the **Firebase Local Emulator Suite** before production deployment.
2. **Deterministic Batch Pipeline & Recovery:** The GitHub Actions batch processor must be tested against simulated GitHub API failures, rate limits, corrupt assets, lease race conditions, full-history commit recoveries, and deployment aborts using mocked HTTP fixtures.
3. **Cross-Browser Vanilla Compliance:** Frontend modules and CSS components must operate natively across Chromium, Gecko, and WebKit rendering engines without requiring transpilers or polyfills.

---

## 2. Testing Pyramid & Layer Breakdown

```mermaid
flowchart TD
    subgraph Pyramid["GitStore Testing Hierarchy"]
        E2E["E2E Lifecycle Tests\n(Submission -> Verification -> Two-Phase Deploy -> UI Render)"]
        SecRules["Firestore Security Rules Tests\n(Firebase Local Emulator Suite + @firebase/rules-unit-testing)"]
        WorkerInteg["Batch Worker Integration Tests\n(Mocked GitHub REST API, Full-History Reconciler, Step-Up Verification, Digest Validation)"]
        Unit["Frontend & Utility Unit Tests\n(Sanitizers, Formatters, Semver, URL Parsers, Digest Hasher)"]
    end

    Unit --> WorkerInteg
    WorkerInteg --> SecRules
    SecRules --> E2E
```

---

## 3. Layer 1: Frontend & Utility Unit Testing

- **Runner:** Node.js native test runner (`node --test`).
- **Scope:**
  - `sanitizers.js`: Verify that malicious SVG tags, `<script>` tags, and `javascript:` URIs are completely stripped via DOMPurify.
  - `formatters.js`: Validate byte size formatting (`1024 B -> 1.0 KB`, `1048576 B -> 1.0 MB`), date formatting, and semver comparison.
  - `url-validator.js`: Confirm regex correctly matches valid GitHub repository URLs (`https://github.com/owner/repo`) and rejects phishing or spoofed domains.
  - `catalog-digest.js`: Assert deterministic SHA-256 digest computation over canonicalized sorted JSON files.

---

## 4. Layer 2: Firestore Security Rules Testing (Local Emulator)

- **Tool:** `@firebase/rules-unit-testing` running against the official **Firebase Firestore Emulator**.
- **Execution:** Runs in local development and inside CI workflows prior to deployment.
- **Key Test Cases:**
  1. **Anonymous Read on Public Data:** Anonymous client CAN read published active products (`/products`), categories (`/categories`), and public developer profiles (`/public_profiles`).
  2. **Private User Document Lockout (`/users/{uid}`):**
     - Anonymous client CANNOT read `/users/{uid}`.
     - Authenticated developer CANNOT read another developer's `/users/{otherUid}` document.
     - Authenticated developer CAN read their own `/users/{uid}` document.
     - Authenticated developer CANNOT modify privileged fields (`isAdmin`, `isVerifiedDeveloper`, `reputationScore`, `githubNumericId`, `email`, `emailVerified`).
  3. **Public Profile Separation (`/public_profiles/{uid}`):**
     - Any user CAN read `/public_profiles/{uid}`.
     - Owner CAN update safe fields (`displayName`, `avatarUrl`, `bio`).
     - Owner CANNOT modify worker-managed badges (`isVerifiedDeveloper`, `publishedProductIds`).
  4. **Internal Collection Default-Deny:**
     - Client (anonymous or authenticated) CANNOT read or write `/system_deployments/{generationId}`.
     - Client CANNOT read or write `/system_config/deployment_state`.
     - Client CANNOT read or write `/system_config/catalog_state`.
     - Client CANNOT read or write `/admin_logs/{logId}`.
     - Client CAN read `/system_config/global_settings`, but CANNOT write.
  5. **Submission Tampering Protection:** Authenticated developer CANNOT update a submission that is already in `processing`, `approved`, or `published` status, and CANNOT alter ownership fields (`developerUid`, `claimStatus`, `ownershipProof`).
  6. **Claim Request Protection & Whitelist:** Authenticated user CAN create a `claim_requests` doc with their own `auth.uid` and `status: 'pending'`, but CANNOT inject privileged or worker fields (`developerUid`, `verifiedGithubId`, `claimStatus`, `isAdmin`, `workerLease`, `workerId`).
  7. **Moderation Queue Security & Mandatory Step-Up Freshness:**
     - Request write FAILS if token `auth_time` is older than 300 seconds (`(request.time - auth_time) >= 300`).
     - Request write SUCCEEDS if `auth_time` is fresh (< 300s), fields match strict whitelist, and `stepUpProof` carries valid schema.
     - Client CANNOT update or delete moderation requests once written.

---

## 5. Layer 3: Batch Worker Integration & Concurrency Testing

- **Scope:** Verify the GitHub Actions processing engine (`scripts/processor/index.js`).
- **Mock Framework:** Mock the GitHub REST API using `nock` or `msw`.
- **Key Test Scenarios:**

### 5.1 Root Administrator Bootstrap & Collaborator Permission Mapping
1. **Numeric ID Normalization Unit Tests (`normalizeGithubId`):**
   ```javascript
   assert.strictEqual(normalizeGithubId(12345678), "12345678");
   assert.strictEqual(normalizeGithubId("12345678"), "12345678");
   assert.strictEqual(normalizeGithubId("  12345678  "), "12345678");
   assert.strictEqual(normalizeGithubId(null), null);
   assert.strictEqual(normalizeGithubId(undefined), null);
   assert.strictEqual(normalizeGithubId(""), null);
   assert.strictEqual(normalizeGithubId("abc123"), null);
   assert.strictEqual(normalizeGithubId("12.34"), null);
   ```
2. **Root Administrator Bootstrap Test:**
   - Mock `GET /repos/Rohitpvt/GitStore` returning `{ owner: { id: 12345678, login: "Rohitpvt" } }`.
   - Mock Firebase Admin SDK `admin.auth().verifyIdToken()` returning `uid: "admin_uid"`.
   - Mock `admin.auth().getUser("admin_uid")` returning `providerData: [{ providerId: "github.com", uid: "12345678" }]`.
   - Assert: Caller is authenticated as ROOT ADMINISTRATOR directly without making collaborator endpoint queries.
3. **Collaborator Permission Mapping Test Fixtures (Delegated Maintainers):**
   ```javascript
   const fixtures = [
     { resp: { permission: 'admin', role_name: 'admin' }, expected: 'AUTHORIZED' },
     { resp: { permission: 'write', role_name: 'maintain' }, expected: 'AUTHORIZED' },
     { resp: { permission: 'write', role_name: 'write' }, expected: 'FORBIDDEN' },
     { resp: { permission: 'read', role_name: 'triage' }, expected: 'FORBIDDEN' },
     { resp: { permission: 'read', role_name: 'read' }, expected: 'FORBIDDEN' },
     { resp: { permission: 'write', role_name: 'custom_reviewer' }, expected: 'FORBIDDEN' },
     { resp: { permission: 'admin' }, expected: 'AUTHORIZED' },
     { resp: {}, expected: 'FORBIDDEN' },
   ];
   ```

### 5.2 Mandatory Step-Up Reauthentication & Token Destruction Tests
- **Valid Step-Up Flow:**
  - Mock GitHub API `GET /user` with ephemeral token returning `id: 12345678`.
  - Caller linked GitHub ID is `12345678`.
  - Assert: Worker marks request `processing`, deletes `ephemeralGithubToken` (`FieldValue.delete()`), records `consumedAt`, and authorizes execution.
- **Identity Mismatch Rejection:**
  - Ephemeral token belongs to GitHub ID `99999999`, but caller linked ID is `12345678`.
  - Assert: Worker aborts with security alarm, marks request `rejected`, and purges token.
- **Replay Protection Test:**
  - Second worker cycle attempts to process a request already containing `consumedAt`.
  - Assert: Worker fails closed and rejects request as replayed attempt.

### 5.3 Full-History Commit Discovery & Branch Ancestry Tests
- **Discovery Beyond 10-Commit Window:**
  - Simulate commit history where candidate manifest commit is at depth 15 (`origin/main~15`).
  - Worker runs unshallow `git log --format="%H" origin/main -- data/deployment-manifest.json`.
  - Assert: Worker recovers commit SHA at depth 15 where legacy `git log -n 10` failed.
- **Branch Ancestry Assertion (`git merge-base --is-ancestor`):**
  - Simulate candidate commit from an unmerged feature branch or fork.
  - Assert `git merge-base --is-ancestor ${candidateSha} origin/main` exits non-zero; reconciler rejects candidate commit.
- **Deterministic Disambiguation via Catalog Digest:**
  - Two candidate commits match `generationId` and `generationSequence`.
  - Worker computes `catalogDigest` on both commits.
  - Assert: Worker selects the latest commit on `origin/main` whose `catalogDigest` matches `gen.catalogDigest`.

### 5.4 Missing Pages Deployment ID Handling (Cases A, B, C, D)
- **Case A Test:** Pages deployment exists and succeeded on GitHub Pages API -> capture ID, verify CDN, finalize Firestore.
- **Case B Test:** No remote Pages deployment found for the commit -> worker packages `site/` and dispatches deployment.
- **Case C Test:** Remote Pages deployment failed -> retry same commit up to 3 times before preserving in `deployment_retained`.
- **Case D Test:** Remote deploy succeeded, DB finalization interrupted -> idempotent finalization without re-deploying.

### 5.5 Durable Catalog Revisions & Work Preservation Tests
- Increment `latestRequestedRevision` in `/system_config/catalog_state` while compilation worker is running.
- Post-deployment, assert worker detects `latestRequestedRevision > activeCapturedRevision` (`isDirty: true`) and triggers follow-up compilation.

### 5.6 Split-Job Workflow & Isolated Staging Tests
- Verify `build-and-commit` job has `contents: write` but zero Pages permissions.
- Verify `deploy-to-pages` job has `pages: write`, `id-token: write` but zero `contents: write` permissions.
- Verify `actions/upload-pages-artifact@v5` packages exclusively from `site/` and that internal files (`.git`, `.github`, `docs/`, `scripts/`) are excluded.

---

## 6. Layer 4: Storefront Client Integration & Performance Testing

- **Testing Tools:** Playwright / Chromium headless.
- **Assertions:**
  1. **Catalog Rendering:** `index.html` loads and renders 50 product cards from `data/catalog.json` in under 300ms.
  2. **Filtering Performance:** Clicking category or platform filters updates the DOM in under 50ms without network round-trips.
  3. **Search Responsiveness:** Typing in the search input highlights matching applications instantaneously.
  4. **Accessibility Audit:** Automated axe-core scan confirms zero WCAG 2.1 AA violations.
  5. **Download Link Integrity:** Verifies that clicking "Download" directs user directly to the upstream GitHub Release asset URL.
