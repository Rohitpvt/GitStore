# GitStore — Comprehensive Testing Strategy

**Document Version:** 1.5.0  
**Phase:** Phase 0.1.4 — Evidence Alignment, Deployment Recovery & Governance Consistency  
**Status:** Approved Quality Assurance & Testing Framework (Hardened & Reconciled)  

---

## 1. Testing Philosophy & Quality Invariants

GitStore adheres to a layered testing hierarchy designed to guarantee platform reliability, client security, and pipeline idempotency without incurring external testing costs.

### Testing Invariants
1. **Zero-Trust Security Verification:** All Firestore Security Rules must be proven secure using automated unit tests executed against the **Firebase Local Emulator Suite** before production deployment.
2. **Deterministic Batch Pipeline:** The GitHub Actions batch processor must be tested against simulated GitHub API failures, rate limits, corrupt assets, lease race conditions, and deployment aborts using mocked HTTP fixtures.
3. **Cross-Browser Vanilla Compliance:** Frontend modules and CSS components must operate natively across Chromium, Gecko, and WebKit rendering engines without requiring transpilers or polyfills.

---

## 2. Testing Pyramid & Layer Breakdown

```mermaid
flowchart TD
    subgraph Pyramid["GitStore Testing Hierarchy"]
        E2E["E2E Lifecycle Tests\n(Submission -> Verification -> Two-Phase Deploy -> UI Render)"]
        SecRules["Firestore Security Rules Tests\n(Firebase Local Emulator Suite + @firebase/rules-unit-testing)"]
        WorkerInteg["Batch Worker Integration Tests\n(Mocked GitHub REST API, Lease Races, Reconciler & Two-Phase Publish)"]
        Unit["Frontend & Utility Unit Tests\n(Sanitizers, Formatters, Semver, URL Parsers)"]
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

---

## 4. Layer 2: Firestore Security Rules Testing (Local Emulator)

- **Tool:** `@firebase/rules-unit-testing` running against the official **Firebase Firestore Emulator**.
- **Execution:** Runs in local development and inside CI workflows prior to deployment.
- **Key Test Cases:**
  1. **Anonymous Read:** Anonymous client CAN read published active products and categories.
  2. **Anonymous Write:** Anonymous client CANNOT write to `submissions`, `moderation_requests`, `claim_requests`, or `users`.
  3. **Self-Escalation Attack:** Authenticated developer CANNOT update their user document to set `isAdmin: true` or `isVerifiedDeveloper: true`.
  4. **Submission Tampering:** Authenticated developer CANNOT update a submission that is already in `processing`, `approved`, or `published` status, and CANNOT alter ownership fields (`developerUid`, `claimStatus`, `ownershipProof`).
  5. **Claim Request Protection:** Authenticated user CAN create a `claim_requests` doc with their own `auth.uid` and `status: 'pending'`, but CANNOT update, delete, or create a claim request with another user's UID.
  6. **Claim Request Field Injection Attack:** Authenticated user CANNOT inject privileged or worker fields (`developerUid`, `verifiedGithubId`, `claimStatus`, `isAdmin`, `processedAt`, `reviewedAt`, `reviewedBy`, `resolution`, `rejectionReason`, `workerLease`, `workerId`) into `/claim_requests`. Any unwhitelisted key triggers an immediate `PERMISSION_DENIED` rule violation.
  7. **Moderation Queue Security & Whitelisting:** Authenticated user CAN create a `moderation_requests` doc with `status: 'pending'`, but CANNOT update or delete it, and CANNOT inject unwhitelisted keys.
  8. **Unauthorized Publish:** Authenticated developer CANNOT create a document directly in the `products` or `system_deployments` collections.

---

## 5. Layer 3: Batch Worker Integration & Concurrency Testing

- **Scope:** Verify the GitHub Actions processing engine (`scripts/processor/index.js`).
- **Mock Framework:** Mock the GitHub REST API using `nock` or `msw`.
- **Key Test Scenarios:**

### 5.1 Root Administrator Bootstrap & Collaborator Permission Mapping
Test root admin bootstrap identity assertion, numeric ID normalization, and delegated maintainer fixtures:
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
     { resp: { permission: 'admin' }, expected: 'AUTHORIZED' }, // Legacy admin without role_name
     { resp: {}, expected: 'FORBIDDEN' },                      // Empty response
   ];
   ```

### 5.2 Token Permission Classification
- *Verification Status:* **PENDING INTEGRATION TEST**.
- Assert that if the worker receives `403 Resource not accessible by integration` using `secrets.GITHUB_TOKEN`, it automatically falls back to `secrets.ADMIN_VERIFY_TOKEN` (fine-grained PAT) and successfully authenticates.

### 5.3 Globally Unique Entity ID Collision Tests
- **Cross-Repository Same-Slug Collision:**
  - Repo 1 (Numeric ID: `892341901`) with product slug `viewer` -> IDs generated: `prod_892341901_viewer`.
  - Repo 2 (Numeric ID: `945112002`) with product slug `viewer` -> IDs generated: `prod_945112002_viewer`.
  - Assert both products coexist in Firestore without overwriting or key collisions.
- **Issue Submission Collision:**
  - Repo 1 issue #42 -> `sub_issue_892341901_42`.
  - Repo 2 issue #42 -> `sub_issue_945112002_42`.
  - Assert submission keys are globally unique.

### 5.4 Publication State Machine, Non-Circular Manifest & Dedicated Pages API Tests
- **Non-Circular Manifest Content Check:**
  - Compile static catalog and inspect generated `data/deployment-manifest.json`.
  - Assert manifest contains `generationId`, `generationSequence`, `generatedAt`, `productCount`, and `environment`.
  - Assert manifest does **NOT** contain `commitSha` and does **NOT** contain `deployedAt`.
- **Post-Commit Firestore Failure Recovery Test:**
  - Simulate pushed Git commit on `origin/main` containing manifest `{ generationId: "gen_test_02", generationSequence: 105 }`, but Firestore `/system_deployments/gen_test_02.catalogCommitSha` is null.
  - Reconciler executes `git log -n 10` and `git show` blob inspection.
  - Assert: Reconciler extracts matching commit SHA, atomically writes `catalogCommitSha` to Firestore, and resumes deployment idempotently.
  - Simulate remote search finding no matching commit -> generation marked `status: 'failed'`, `failureReason: 'commit_not_found_on_remote'`, submissions cleanly coalesced into next run.
- **Duplicate Sequence Conflict Test (Case 3):**
  - Set `latestPublishedSequence = 105`, `latestPublishedGenerationId = "gen_first"`.
  - Simulate a competing worker attempting finalization with `generationSequence = 105`, `generationId = "gen_second"`.
  - Assert: Reconciler triggers Case 3, fails closed, logs `DEPLOYMENT_SEQUENCE_CONFLICT` alarm in `/admin_logs`, and aborts without mutating publication state.
- **Dedicated GitHub Pages API Status Test:**
  - Mock dedicated Pages deployment endpoint `GET /repos/Rohitpvt/GitStore/pages/deployments/101` returning `{ "status": "succeed" }`.
  - Assert worker correctly identifies Pages-specific success status `"succeed"`.
  - Mock endpoint returning `{ "status": "failed" }` and assert generation record is preserved in `/system_deployments` without reverting products to `approved`.
- **Monotonic Generation Ordering Assertion:**
  - Set `latestPublishedSequence = 104` in Firestore.
  - Simulate a stale worker attempting to finalize generation with `generationSequence = 103`.
  - Assert worker aborts finalization to prevent catalog regression.
- **Durable Failed Generation Preservation:**
  - Simulate Pages deployment failure for generation `gen_test_01` (commit `abc1234`).
  - Assert `/system_deployments/gen_test_01` document is preserved with `attemptHistory`, `lastObservedPagesStatus: 'failed'`, and `retryEligibility: true`.
  - Assert product documents remain associated with `gen_test_01` and are NOT reverted to `approved`.

### 5.5 Secure Submission Claiming (`/claim_requests`)
- **Valid Claim:** User with verified `github.com` provider ID `'583231'` claims submission with `submitterGithubId: '583231'`. Assert `developerUid` is set and status is `claimed`.
- **Identity Mismatch:** User with provider ID `'999999'` attempts to claim submission `'583231'`. Assert claim request is marked `rejected_mismatch`, submission remains `unclaimed`.
- **Duplicate Claim:** Assert claiming an already-claimed submission is rejected with `rejected_already_claimed`.

### 5.6 GitHub Actions Event and Fork Security Tests
- Assert privileged workflows specify only `schedule` and `workflow_dispatch` triggers on default branch.
- Assert `pull_request_target` and `workflow_run` are absent from all repository workflow files.
- Assert pull request validation workflows have zero access to repository secrets.

---

## 6. Layer 4: Storefront Client Integration & Performance Testing

- **Testing Tools:** Playwright / Chromium headless.
- **Assertions:**
  1. **Catalog Rendering:** `index.html` loads and renders 50 product cards from `data/catalog.json` in under 300ms.
  2. **Filtering Performance:** Clicking category or platform filters updates the DOM in under 50ms without network round-trips.
  3. **Search Responsiveness:** Typing in the search input highlights matching applications instantaneously.
  4. **Accessibility Audit:** Automated axe-core scan confirms zero WCAG 2.1 AA violations.
  5. **Download Link Integrity:** Verifies that clicking "Download" directs user directly to the upstream GitHub Release asset URL.
