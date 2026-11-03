# GitStore — Submission Pipeline & Batch Processing Engine

**Document Version:** 1.7.0  
**Phase:** Phase 0.1.6 — Step-Up Proof Integrity, Firestore Rules & Deployment Reconciliation  
**Status:** IN_REVIEW — Awaiting Explicit User Approval  
**Execution Engine:** GitHub Actions (`.github/workflows/batch-processor.yml`)

---

## 1. Overview & Core Invariants

The GitStore Submission Pipeline is the ingestion, verification, and static publishing engine of the platform. It bridges developer submissions with storefront delivery under strict security, eventual consistency, and zero-cost constraints.

### Guiding Pipeline Invariants
1. **Globally Unique Entity Decoupling:** Multiple independent software products can originate from a single GitHub repository. Products are identified by `prod_{repoNumericId}_{productSlug}` and repositories by `gh_repo_{repoNumericId}`. Collisions across repositories with identical product slugs are mathematically impossible.
2. **Unified Ingestion Without Fake Accounts:** Submissions from the Developer Dashboard and GitHub Issue Forms feed the same queue. Issue form submitters do **not** receive placeholder Firebase accounts; their submissions are stored with `submitterGithubId` and `claimStatus: 'unclaimed'`, claimed securely via an asynchronous `/claim_requests` queue when they log in.
3. **Rigorous Ownership & Authority Tiers:** Only Direct Personal Repository Owners (`repo.owner.id == submitter.githubId`) possess inherent release authority to qualify for automated publication. Organization members, committers, and community nominators lack proven release authority and are strictly gated to manual administrator review (`needs_review`).
4. **Distinction of Verified Developer Status:** Personal repository ownership does **not** automatically award the platform "Verified Developer" badge. Auto-published products from unverified personal owners carry standard community attribution.
5. **Formal 12-State Recoverable Deployment State Machine:** Consistency between GitHub Pages and Cloud Firestore is governed by a formal 12-state state machine. The publication manifest (`data/deployment-manifest.json`) records `generatedAt` and a deterministic SHA-256 `catalogDigest` (computed over canonicalized catalog files, excluding manifest), strictly omitting circular commit SHAs. Full-history commit discovery on unshallow fetch (`git log origin/main`) with branch ancestry verification (`git merge-base --is-ancestor`) and paginated GitHub Commits API resolves post-commit failures. Missing Pages deployment IDs are deterministically resolved across Cases A, B, C, and D via official endpoint `GET /repos/{owner}/{repo}/pages/deployments/{catalogCommitSha}`.
6. **Least-Privilege Split-Job Pages Workflow & Cross-Job Handoff:** The batch workflow is split into two sequential jobs: `build-and-commit` (granted `contents: write`, `metadata: read`) and `deploy-to-pages` (granted `pages: write`, `id-token: write`, `contents: read`, `metadata: read` targeting `environment: github-pages`). Cross-job outputs explicitly pass parameters (`generation_id`, `generation_sequence`, `catalog_commit_sha`, `catalog_digest`, `deploy_needed`). Public assets are staged strictly into an isolated `site/` directory (`actions/upload-pages-artifact@v3` packaging `path: site/`).
7. **Pre-Deployment Generation Ordering Guard:** Immediately prior to invoking `actions/deploy-pages@v4`, the workflow executes a 4-case sequence assertion against Firestore, aborting stale or conflicting runs before Pages deployment can occur.
8. **Durable Catalog Revisions & Transactional Work Preservation:** A durable revision counter in `/system_config/catalog_state` guarantees that approved submissions arriving while a compilation or deployment is underway are never lost. Monotonic advancement is enforced inside an atomic Firestore transaction.

---

## 2. Recoverable 12-State Submission & Deployment State Machine

```mermaid
stateDiagram-v2
    [*] --> queued: Submitted via Dashboard or Issue Form
    queued --> processing: Leased by Actions Worker (Limit 3, 15m Lease)
    
    state processing {
        [*] --> VerifyOwnership: Check Direct Owner vs Org/Committer/Community
        VerifyOwnership --> ValidatingRepo: Check Public, Not Archived
        ValidatingRepo --> ValidatingRelease: Verify Release Tag Exists
        ValidatingRelease --> ValidatingAsset: Check File Extension & Exact Size
    }

    processing --> failed: Automation / API Failure (404, Deleted Tag)
    processing --> rejected: Blacklisted, Malformed, Policy Violation
    
    processing --> needs_review: Org Member OR Committer OR Community
    processing --> approved: Direct Personal Owner (Valid Assets)
    
    needs_review --> approved: Admin Approves via moderation_requests (With Action-Bound Proof)
    needs_review --> rejected: Admin Rejects Listing with Feedback
    
    approved --> catalog_generating: Leased by Static Compilation Worker
    catalog_generating --> compilation_failed: Static JSON Generation Throws Error
    catalog_generating --> stale_aborted: Monotonic Sequence Pre-Check Detects Stale Run
    catalog_generating --> catalog_committed: Catalog & Manifest Committed to Git
    catalog_committed --> deploying: In-Workflow actions/deploy-pages Dispatched
    deploying --> deployment_success: Dedicated Pages API Reports status 'built'
    deployment_success --> deployment_verified: Public HTTP Manifest & Digest Verified
    deployment_verified --> published: Firestore Document Finalized to published
    deploying --> deployment_failed: Deploy Times Out / Errored
    deployment_failed --> deployment_retained: Record Preserved in /system_deployments
    deployment_retained --> reconciliation_active: Leased by Startup Reconciler
    reconciliation_active --> deploying: Cases B/C Re-Dispatch Deployment
    reconciliation_active --> deployment_verified: Cases A/D Verified Live State
    reconciliation_active --> reconciliation_failed: Split-Brain or Unrecoverable Divergence
    published --> [*]
    failed --> queued: Developer Corrects Metadata & Resubmits
    rejected --> [*]
```

---

## 3. Ingestion Channels & Identity Linking

### Channel A: Developer Web Dashboard
- Authenticated via GitHub OAuth (`auth.uid` + verified numeric GitHub ID in `providerData`).
- Document created directly in `/submissions/{submissionId}` with `claimStatus: 'claimed'` and `developerUid: auth.uid`.

### Channel B: GitHub Issue Forms
- Author identity: `issue.user.id` (numeric string) and `issue.user.login` from trusted GitHub webhook payload.
- **Zero Fake Accounts:** Worker registers submission with `developerUid: null`, `submitterGithubId: issue.user.id`, and `claimStatus: 'unclaimed'`.
- Document ID: `sub_issue_{repoNumericId}_{issueNumber}`.

### 3.1 Secure Asynchronous Claiming Flow (`/claim_requests`)
1. Developer authenticates on GitStore via GitHub OAuth (or Email Magic Link + GitHub Link).
2. Developer initiates claim on `dashboard.html`.
3. Client writes to `/claim_requests/{requestId}`:
   ```typescript
   {
     requestId: string;
     requesterUid: string;          // request.auth.uid
     targetSubmissionId: string;
     status: 'pending';
     createdAt: serverTimestamp();
   }
   ```
4. **Strict Client Whitelist & Security Rules Guardrails:**
   Clients must **never** be able to inject worker-managed or privileged fields. Security Rules enforce:
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
     allow update, delete: if false; // Privileged worker only
   }
   ```
   **Rejected Client Fields:** Any request attempting to inject `developerUid`, `verifiedGithubId`, `claimStatus`, `isAdmin`, `processedAt`, `reviewedAt`, `reviewedBy`, `resolution`, `rejectionReason`, `workerLease`, or `workerId` is immediately rejected.
5. Worker reads pending claim requests during batch run:
   - Queries Firebase Admin SDK: `admin.auth().getUser(req.requesterUid)`.
   - Finds `userRecord.providerData.find(p => p.providerId === 'github.com')`. Fail closed if absent.
   - Extracts `verifiedGithubId = provider.uid`.
   - Checks target submission: `submission.submitterGithubId === verifiedGithubId`.
   - Executes atomic Firestore transaction:
     - Checks `submission.developerUid == null` (prevents competing claims).
     - Sets `submission.developerUid = req.requesterUid`
     - Sets `submission.claimStatus = 'claimed'`
     - Sets `submission.claimedAt = serverTimestamp()`
     - Sets `claimReq.status = 'completed'`
     - Appends log to `/admin_logs`
   - Rejects with `rejected_mismatch` if IDs do not match, or `rejected_already_claimed` if claimed. Handles duplicates idempotently.

### 3.2 Ownership Verification Proof Matrix
1. **Tier 1 (Direct Personal Owner):** `repo.owner.id == submitter.githubId`. Possesses inherent release authority. Auto-publishes on valid asset checks.
2. **Tier 2 (Organization Member):** `GET /orgs/{org}/public_members/{login}` returns 204. Proves organization association, but not repo release authority. Forced to `needs_review`.
3. **Tier 3 (Committer):** `GET /repos/{owner}/{repo}/commits?author={login}` returns merged commits. Proves contribution, but not release authority. Forced to `needs_review`.
4. **Tier 4 (Community Nomination):** Submitter has no commits or ownership. Forced to `needs_review`. Never eligible for auto-publication.

---

## 4. Globally Unique Multi-Product Entity Hierarchy

To support repositories containing multiple distributable applications without cross-repository slug collisions:

```text
GitHub Repository: https://github.com/microsoft/sysinternals (Repo ID: 892341901)
  ├── Repository Doc ID: gh_repo_892341901
  ├── Product 1: Process Explorer
  │     ├── Product ID: prod_892341901_procexp
  │     ├── Slug: procexp
  │     └── Download Asset: ProcExp64.exe (Asset ID: 9812401)
  └── Product 2: Autoruns
        ├── Product ID: prod_892341901_autoruns
        ├── Slug: autoruns
        └── Download Asset: Autoruns64.exe (Asset ID: 9812402)
```

- **Cross-Repository Safety:** If repository `945112002` also creates a product with slug `viewer`, its product ID is `prod_945112002_viewer`, preventing any collision with `prod_892341901_viewer`.
- **Repository Rename & Transfer Resiliency:** Because GitHub's numeric repository ID never changes upon rename (`user/repo` -> `user/renamed-repo` or transfer to another owner), document keys remain permanently immutable. Upstream metadata (`repoOwner`, `repoName`) is updated dynamically.

---

## 5. Recoverable Publication Protocol & Verification

```mermaid
flowchart TD
    A[Worker Starts Batch Cycle] --> B[Step 0: Run reconcilePublishedState]
    B --> C{Live deployment verified for unconfirmed generations?}
    C -->|Yes| D[Heal Firestore: Finalize status to published, advance sequence & revision]
    C -->|No| E[Step 1: Process Pending claim_requests & moderation_requests]
    D --> E
    E --> F[Step 2: Claim Queued Submissions Limit 3]
    F --> G[Step 3: Run GitHub Verification Suite]
    G --> H{Any Submissions Transition to approved or revision dirty?}
    H -->|No| I[Exit Run]
    H -->|Yes| J[Step 4: Transition to catalog_generating & Capture Revision Snapshot]
    J --> K[Step 5: Generate static JSON, compute recursive catalogDigest, write manifest]
    K --> L[Step 6: Stage public assets into site/ directory]
    L --> M[Step 7: Git Pull --rebase, Commit & Push origin main]
    M --> N[Step 8: Capture catalogCommitSha via git rev-parse HEAD & Record in system_deployments]
    N --> O[Step 9: Output cross-job parameters & Pre-Deploy Guard]
    O --> P[Step 10: Split Job deploy-to-pages: upload-pages-artifact & deploy-pages]
    P --> Q{Dedicated Pages API Reports status: 'built'?}
    Q -->|Yes| R[Step 11: Public HTTP Manifest & Digest Verification on Pages CDN]
    R -->|Verified & Monotonic| S[Step 12: Finalize Submissions to published, advance latestPublishedSequence & revision]
    R -->|Cache Propagating| T[Leave in system_deployments as deploying; Reconciler will finalize]
    Q -->|Failed / Timeout| U[Record failure in system_deployments; Retain catalogCommitSha for retry]
    S --> V[Release Leases, Log admin_logs & Exit]
    T --> V
    U --> V
```

### 5.1 Deployment Verification, Concurrency & Monotonic Ordering Details

#### 1. Non-Circular Manifest Architecture & Recursive Canonical `catalogDigest`
The static catalog publication creates `data/deployment-manifest.json` containing:
```json
{
  "generationId": "gen_1730300400000_a1b2c3d4",
  "generationSequence": 104,
  "generatedAt": "2026-11-03T09:00:00.000Z",
  "catalogDigest": "sha256:v1:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "productCount": 42,
  "schemaVersion": "1.7.0",
  "environment": "github-pages"
}
```
- **Recursive Canonical Digest Calculation:** Computed over static files (`data/catalog.json`, `data/categories.json`, and all `data/products/*.json`) with recursive Unicode key sorting, array preservation, and length-prefixed null-byte framed boundaries (`FILE\0${relPath}\0${byteLength}\0`). The manifest is excluded from computation.
- **Strict Invariant:** The manifest does **not** contain the SHA of the Git commit that contains it. It records `generatedAt` rather than `deployedAt`. Actual deployment timestamp is recorded post-deployment in Firestore (`verifiedAt`, `finalizedAt`, `lastPublishedAt`).

#### 2. Dedicated GitHub Pages Deployment Endpoint Check
Deployments are verified via the official GitHub Pages deployments API:
`GET https://api.github.com/repos/Rohitpvt/GitStore/pages/deployments/{catalogCommitSha}`
- Returns `{ "status": "built" }` upon verified completion. (Distinct from generic Deployments API `state: "success"`).
- Non-terminal statuses include `"in_progress"` and `"queued"`.
- Error status is `"errored"`.

#### 3. Least-Privilege Split-Job Pages Configuration & Isolated Staging
- **Job 1 (`build-and-commit`):** `contents: write`, `metadata: read`. Compiles catalog, verifies digest, stages files into `site/`, commits and pushes to `origin/main`. Outputs parameters for Job 2.
- **Job 2 (`deploy-to-pages`):** `pages: write`, `id-token: write`, `contents: read`, `metadata: read`. Runs pre-deploy guard, packages `site/` staging directory, executes deployment to `environment: github-pages`.
- **Verified Action Releases (Pinned Commit SHAs):** `actions/checkout@v4`, `actions/setup-node@v4`, `actions/upload-pages-artifact@v3`, `actions/deploy-pages@v4`.

#### 4. GitHub Actions Concurrency & Durable Coalescing
- **Actions Runner Queue & Environment Concurrency Boundary:**
  Pages deployment is bound to the repository concurrency group:
  ```yaml
  concurrency:
    group: 'github-pages'
    cancel-in-progress: false
  ```
- **Durable Coalescing (Work Preservation Guarantee):**
  When compiling, the batch processor queries all approved submissions:
  `db.collection('submissions').where('status', '==', 'approved').get()`
  All approved items are included in the new generation, preventing work loss even if intermediate runner jobs were discarded.

#### 5. Four-Case Generation Ordering & Idempotency Matrix:
Every deployment generation is checked against `/system_config/deployment_state.latestPublishedSequence`:
1. *Case 1: Stale Generation (`seq < latestPublishedSequence`):* Rejected immediately. Deployment and finalization aborted to prevent catalog rollback.
2. *Case 2: Idempotent Retry (`seq === latestPublishedSequence && id === latestPublishedGenerationId`):* Verified and finalized without re-compiling or re-committing catalog.
3. *Case 3: Duplicate Sequence Conflict (`seq === latestPublishedSequence && id !== latestPublishedGenerationId`):* Critical fail-closed condition. Two distinct generation IDs claiming the same sequence number. Deployment halted; high-severity audit alarm logged in `/admin_logs`; operator intervention required.
4. *Case 4: Valid New Generation (`seq > latestPublishedSequence`):* Valid progression. Concurrency lease prevents overlap; proceeds to Pages deploy and sequence advance.

---

### 5.2 Comprehensive 11 Failure Scenarios & Idempotent Recovery Matrix

| # | Failure Scenario | Detection Mechanism | Idempotent Recovery & Consistency Resolution |
|---|---|---|---|
| **1** | **Catalog generation failure** | Node.js process throws syntax error or JSON compilation exception prior to commit. | Process aborts. Zero Git commits created. Submissions remain in `approved`. A failed generation record is logged with `stage: 'generation'`. Submissions are cleanly picked up on next batch cycle. |
| **2** | **Catalog commit failure** | Git push rejected due to concurrent push or branch protection conflict. | Process executes `git reset --hard origin/main`. Submissions remain in `approved`. Zero invalid generation records persisted. Re-compiled on next cycle with updated `origin/main`. |
| **3** | **Post-commit Firestore failure (Missing commit SHA)** | Git push to `origin/main` succeeds, but runner terminates or network fails *before* worker can persist `catalogCommitSha` in Firestore `/system_deployments/{generationId}`. | **Full-History Discovery & Digest Match:** Reconciler executes unshallow full-history search: `git log --format="%H" origin/main -- data/deployment-manifest.json` and paginated GitHub Commits API loop. For each candidate SHA, reconciler cryptographically asserts branch ancestry (`git merge-base --is-ancestor ${sha} origin/main`), reads blob directly via `git show ${sha}:data/deployment-manifest.json`, parses JSON, and matches `generationId`, `generationSequence`, and deterministic `catalogDigest`. Disambiguates by selecting latest commit on `origin/main` whose digest matches. Atomically saves recovered SHA in `/system_deployments/{generationId}` and resumes Pages deployment. |
| **4** | **Deployment workflow not triggered** | Artifact upload step fails or runner runs out of disk space before Pages deployment starts. | `catalogCommitSha` already exists in Git and is recorded in `/system_deployments/{generationId}` with `status: 'deploying'`. Submissions remain in `catalog_committed` (NOT reverted to `approved`). Reconciler re-executes `actions/upload-pages-artifact@v3` from `site/` for that exact commit SHA. |
| **5** | **Deployment queued but never started** | GitHub Pages deployment infrastructure queued or suspended. | Worker 15-minute lease timer expires. Reconciler checks Pages deployment API. If deployment remains queued or unstarted beyond threshold (30m), reconciler cancels stale deployment and re-dispatches deployment for existing `catalogCommitSha`. |
| **6** | **Pages deployment failure or missing deployment ID** | Dedicated Pages API returns `{ "status": "errored" }`, deployment times out, or `pagesDeploymentId` was unrecorded. | **Cases A-D Handling:** Reconciler queries official Pages deployments API (`GET /repos/Rohitpvt/GitStore/pages/deployments/{catalogCommitSha}`). **Case A:** Remote deployment exists and `status === 'built'` -> capture ID, verify CDN, finalize. **Case B:** Remote API returns 404 (no Pages deployment exists) -> worker restores tree at `catalogCommitSha`, validates digest against `gen.catalogDigest`, rebuilds artifact into `site/`, sets `deploy_needed = true`, and sets `reconciliationStatus = 'pages_dispatched'`. **Case C:** Remote deployment failed -> retry same commit up to 3 times before preserving in `deployment_retained`. **Case D:** Remote deploy succeeded, DB finalization interrupted -> idempotent finalization. |
| **7** | **Pages deploy success, public manifest not yet updated** | Dedicated Pages API reports `status: "built"`, but public HTTP fetch returns HTTP 404 or older manifest due to edge CDN propagation delay. | Worker polls public manifest with exponential backoff (5s, 10s, 20s, up to 5m). If still unpropagated, worker leaves generation in `/system_deployments` with `status: 'deploying'`. The startup reconciler in subsequent batch run verifies propagated manifest and finalizes Firestore. |
| **8** | **Manifest generation mismatch** | Public manifest returns `generationId` different from expected generation. | Evaluated against the 4-case matrix. If `generationSequence < latestPublishedSequence`, wait for CDN edge cache TTL. If `generationSequence > expectedSequence`, newer generation deployed; this generation marked `status: 'stale_aborted'`. If duplicate sequence conflict (`seq === latest && id !== latest`), fail closed. |
| **9** | **Firestore finalization failure** | Network drop or Firestore rate limit while updating submission documents to `status: 'published'`. | Pages is already live! Generation record exists in `/system_deployments` with `catalogCommitSha` and `expectedProductIds`. Startup reconciler verifies live manifest & digest and finishes updating unfinalized submissions to `published`, advancing `catalog_state`. Operation is fully idempotent. |
| **10** | **Worker interruption during finalization** | GitHub Actions runner preempted or terminated midway through batch writes. | 15-minute lease expires. Next batch run detects active generation with unfinalized submissions in `expectedProductIds`. Reconciler completes remaining updates and sets `/system_deployments/{generationId}.status = 'published'`. |
| **11** | **Newer generation appearing while older generation is deploying** | Mutex lease expired and second worker started, or manual dispatch triggered. | Concurrency lease prevents simultaneous runs. If an older run attempts finalization after a newer generation has already published (`generationSequence <= latestPublishedSequence`), monotonic check drops the older update as `stale_aborted`, preventing regression. |

### 5.3 Consistency Guarantee
GitStore provides **guaranteed eventual consistency**: if a deployment succeeds, Firestore will heal to match; if a deployment fails, Firestore will not prematurely expose the product on the storefront or create divergent Git-database states.
