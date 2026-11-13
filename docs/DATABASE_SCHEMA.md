# GitStore — Cloud Firestore Database Schema & Data Architecture

**Document Version:** 1.13.0  
**Phase:** Phase 0.1.13 — Work-Plan Concurrency, Lease Renewal & Reconciliation Correctness  
**Status:** IN_REVIEW — Awaiting Explicit User Approval  
**Data Schema Version:** 1.13.0  
**Database:** Google Cloud Firestore (Native Mode)

---

## 1. Overview & Data Modeling Principles

GitStore employs **Cloud Firestore** as its operational and transactional data backbone. The data model is optimized for document immutability, auditability, fine-grained client security rules, and minimal index overhead.

### Key Data Invariants
1. **Firebase Auth UID as Primary Application Identity & Strict Data Privacy:** All user-centric documents in `users` use the Firebase Auth UID as their document ID (`/users/{uid}`). User records are strictly private (`allow read: if isOwner(uid)`). Public developer profiles are isolated into an explicit public projection (`/public_profiles/{uid}`) with client-restricted, worker-managed reputation badges.
2. **Globally Unique Deterministic Entity IDs:** A single GitHub repository can distribute multiple independent software products. Repositories and products are identified by GitHub's immutable numeric repository ID:
   - Repository: `/repositories/gh_repo_{repoId}` (e.g. `gh_repo_892341901`)
   - Product: `/products/prod_{repoId}_{productSlug}` (e.g. `prod_892341901_viewer`)
   This mathematically prevents collisions even if two repositories under the same owner use the identical slug `viewer`.
3. **No Fake / Placeholder Firebase Accounts:** GitHub Issue submissions do **not** create placeholder Firebase Auth accounts. Unclaimed submissions record `submitterGithubId` (immutable numeric GitHub ID) and `claimStatus: 'unclaimed'`. They are claimed securely via an asynchronous `/claim_requests` queue when the developer authenticates via Firebase.
4. **Step-Up Proof Integrity, ActionHash Checksum & Nonce Replay Protection:** Administrative commands write to `/moderation_requests`. Moderation requests strictly require fresh GitHub step-up reauthentication (`reauthenticateWithPopup`). **Zero raw OAuth access tokens are stored in Firestore.** The request stores an action-bound payload containing caller UID (`requestedByUid`), immutable numeric GitHub ID (`identities['github.com'][0]`), `auth_time`, action, target document path, action nonce, action-specific payload, and a client-generated `actionHash` (which serves as a client data-integrity checksum, not a cryptographic signature). To prevent nonce reuse across different request documents, worker processing atomically claims the request and reserves the nonce in a dedicated collection `/consumed_stepup_nonces/{nonceHash}` within a single Firestore transaction. The Security Rules enforce a strict 5-minute authentication freshness gate at creation time (`auth_time < 300s`), while worker execution operates on a 30-minute queue TTL from `createdAt`. The worker validates the target document path against an explicit action-to-collection mapping and validates action-specific payload schemas.
5. **Node 24 GitHub Actions Pipeline & Provenance-Preserving Staging:** Batch operations run on official GitHub Actions pinned to Node 24 releases (`actions/checkout@v7.0.1`, `actions/setup-node@v7.1.0`, `actions/upload-pages-artifact@v5.0.0`, `actions/deploy-pages@v5.0.1`). Artifact staging follows a single authoritative procedure: the `site/` staging directory is constructed exclusively from the verified source commit, its digest is recomputed and verified (`stagedDigest === catalogDigest`), and no subsequent workflow step copies from `main`. Step outputs are exported via direct append writes to `$GITHUB_OUTPUT`.
6. **12-State Deployment Recovery State Machine & Dedicated Pages Status (`succeed`):** Deployments transition across a formal 12-state state machine. Catalog integrity is proven via a deterministic SHA-256 `catalogDigest` computed recursively over canonicalized catalog files (length-prefixed, null-byte framed `sha256:v1:...`, excluding manifest). GitHub Pages deployment status is verified using the dedicated official endpoint `GET /repos/{owner}/{repo}/pages/deployments/{pages_deployment_id}` (where parameter accepts numeric deployment ID or commit SHA reference) checking for the documented success status `"status": "succeed"` (distinct from generic deployments and builds). In Case B recovery, the reconciler restores the Git tree, validates the digest, reconstructs `site/`, and explicitly exports `deployNeeded = true` to trigger artifact packaging and Pages deployment. Fallback commit recovery follows the full GitHub Commits API pagination loop without arbitrary page caps, distinguishes rate-limit or network errors (`INCOMPLETE_SEARCH`) from true missing commits (`EMPTY_SEARCH`), and validates commit SHA formats before shell execution.
7. **Global Generation Fencing, Concurrency-Safe Work-Plan Preparation, Lease Renewal & 2-Tier Finalization:** When finalizing deployment state, the worker executes a 2-tier process governed by strict generation fences:
   - **Tier 1 Publication Transaction:** Reads current versions of `/system_config/catalog_state`, `/system_config/deployment_state`, and `/system_deployments/{generationId}` BEFORE any writes. Inside the transaction, it verifies that child batch preparation is fully complete (`fanoutPreparationStatus === 'prepared'`), enforces presence of a valid canonical 64-character hex `workPlanDigest` (failing closed as `PREPARED_DIGEST_MISSING_OR_INVALID_ABORT` if missing or malformed), evaluates generation ordering (`evaluateGenerationOrdering`), and enforces catalog revision monotonicity. Stale generations abort immediately (`stale_aborted`) with zero state modifications. Tier 1 publication strictly cannot proceed while preparation is incomplete or owned by another worker.
   - **Tier 2 Submission Fan-Out Execution:** Fan-out work is partitioned into bounded child documents (`/system_deployments/{generationId}/fanout_batches/{batchId}`) capped at 400 IDs each (~55 KB << 1 MiB limit).
     - *Concurrency-Safe Work-Plan Preparation (`prepareFanoutBatches`):* Never relies on stale `genData` passed into the function; always reads the latest generation document inside an atomic preparation ownership transaction (`acquirePreparationLock`). Prevents concurrent workers from preparing different work plans for the same generation via a 15-minute preparation ownership lock (`fanoutPreparationStatus: 'preparing'`, `preparationOwner`, `preparationExpiresAt`). Expired locks are safely reclaimable by subsequent workers. Strictly validates every input submission ID (failing closed as `INVALID_SUBMISSION_ID_ABORT`), deterministically deduplicates and sorts IDs, and computes a canonical 64-character hex `workPlanDigest`. Even on idempotent retries where a generation is already marked `prepared`, it deeply verifies the persisted child batch plan before returning.
     - *Active Batch Content Protection & Immutability:* Never modifies `submissionIds` or `batchDigest` while a batch is actively leased. Existing batch documents are read and asserted; if an active batch contains work-plan content differing from the expected plan, it fails closed as `ACTIVE_BATCH_MUTATION_CONFLICT` (never silently rewriting an active batch). Completed batches (`status === 'completed'`) remain strictly immutable; any digest mismatch fails closed as `COMPLETED_BATCH_CORRUPT_CONFLICT`.
     - *Bounded Cleanup & Paginated Verification:* Stale batch cleanup and deep readback verification use stable cursor pagination (`limit(50)`, `orderBy('batchIndex', 'asc')`, `startAfter(lastDoc)`). Cleanup never removes active or completed batches, and operates idempotently. Deep readback verification is paginated in bounded chunks, ensuring the parent `fanoutPreparationStatus: 'prepared'` marker is persisted ONLY after every expected child batch is cryptographically verified.
     - *Option A Transactional Heartbeat Lease Renewal Protocol (`renewBatchLease`):* Workers acquire an initial 15-minute lease via `acquireBatchLease()`. Before processing each mini-chunk, if less than 5 minutes remain on the lease, the worker transactionally asserts lease ownership (`leaseOwner === workerExecutionId`), verifies unexpired lease status, enforces a 60-minute maximum allowable lease window ceiling from `leaseAcquiredAt` (failing closed as `MAX_LEASE_WINDOW_EXCEEDED`), and extends `leaseExpiresAt` by 15 minutes. Any renewal failure (superseded worker or timeout) causes immediate worker abort (`LEASE_RENEWAL_LOST_ABORT`) without committing batch state.
     - *Fail-Closed Product & Revision Validation:* Missing product documents fail closed (`skipped_product_not_found`). Delisted and flagged products are skipped and never published or reactivated. Mismatched product IDs fail closed. Submissions require a strictly valid positive integer `approvedCatalogRevision <= capturedRevision`; missing, null, malformed, or post-capture revisions fail closed (`skipped_invalid_approval_revision`, `skipped_post_capture_reapproval`).
     - *Atomic Manual-Review Recovery with Deterministic Audit Logging:* Exhausted retries (5 attempts) or fatal sequence conflicts (`HALT_CONFLICT`) trigger a single atomic Firestore transaction (`db.runTransaction`) that reads all affected documents before writes, transitions the active child batch (`status: 'manual_review'`, lease cleared), transitions the parent generation (`submissionFanoutStatus: 'manual_review'`), and writes an immutable audit record to `/admin_logs` using a deterministic document ID (`fanout_mr_${genId}_${batchId}_attempt_${attempts}`). This guarantees idempotency on transaction retries and ensures manual-review recovery cannot succeed without its audit record.
     - *Stable Immutable Reconciler Pagination:* The startup reconciler discovers incomplete published generations via cursor pagination ordered strictly by immutable `generationSequence` ASC (`where('status', '==', 'published')`, `where('submissionFanoutStatus', 'in', ['pending', 'in_progress', 'partial'])`, `orderBy('generationSequence', 'asc')`, `limit(10)`, `startAfter(lastGenDoc)`). Because `generationSequence` is immutable, status field mutations during traversal do not shift cursor positioning. Multi-run scheduled sweeps recover any partial work without skip hazards, providing reliable eventual convergence without claiming false "exactly-once" processing.
8. **Decoupled Storefront Access:** Storefront reads bypass Firestore completely during normal user browsing, consuming static JSON compiled by GitHub Actions and served by GitHub Pages.

---

## 2. Collection Specifications

### 2.1 Collection: `users` (Strictly Private)
- **Path:** `/users/{uid}`
- **Document ID:** Firebase Auth UID (string).
- **Description:** Sensitive profile and authorization metadata for developers and platform administrators. Strictly private to the authenticated owner and batch worker.
- **Access Rule:** Read: Authenticated owner (`request.auth.uid == uid`) and worker via Admin SDK only. Zero public read access. Client write is restricted to safe profile fields via strict creation allowlists and update-field diffing (`diff().affectedKeys().hasOnly(['displayName', 'avatarUrl', 'bio', 'settings', 'updatedAt'])`); client modification of privileged fields (`isAdmin`, `isVerifiedDeveloper`, `reputationScore`, `githubNumericId`, `email`) is blocked.

```typescript
interface UserDocument {
  uid: string;                       // Primary Key (matches Auth UID)
  email: string;                     // Verified contact email (Private PII)
  displayName: string;               // Display handle or GitHub full name
  avatarUrl: string;                 // URL to avatar
  bio?: string;                      // Developer biography
  githubUsername?: string | null;    // Informational handle (e.g. "octocat")
  githubNumericId?: string | null;   // Verified immutable GitHub numeric ID (worker-managed)
  isVerifiedDeveloper?: boolean;     // Admin-granted trust credential (worker-managed only)
  isAdmin?: boolean;                 // Informational flag (worker-managed only)
  reputationScore?: number;          // Calculated score (worker-managed only)
  settings?: {
    notificationEmail?: string;
    emailOnApproval?: boolean;
    emailOnRejection?: boolean;
  };
  createdAt: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.2 Collection: `public_profiles` (Public Read Projection)
- **Path:** `/public_profiles/{uid}`
- **Document ID:** Firebase Auth UID (string).
- **Description:** Public developer profile projection safe for public consumption. Exposes public attribution without leaking emails, GitHub provider UIDs, or internal authorization flags.
- **Access Rule:** Public read (`allow read: if true;`). Authenticated owners may update `displayName`, `avatarUrl`, and `bio`. Fields `isVerifiedDeveloper` and `publishedProductIds` are written strictly by the worker via Admin SDK.

```typescript
interface PublicProfileDocument {
  userId: string;                    // Foreign Key -> users/{uid}
  displayName: string;               // Public display name
  githubUsername: string;            // GitHub handle for link attribution
  avatarUrl: string;                 // Public avatar URL
  bio: string;                       // Public developer bio
  isVerifiedDeveloper: boolean;      // Worker-managed trust badge (client write forbidden)
  publishedProductIds: string[];    // Worker-managed list of published product IDs (client write forbidden)
  createdAt: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.3 Collection: `submissions` (Strict Client Creation Schema)
- **Path:** `/submissions/{submissionId}`
- **Document ID:** Globally unique ID: `sub_issue_{repoId}_{issueNumber}` (issue submissions) or `sub_dash_{nanoid}` (dashboard submissions).
- **Description:** Tracks every submitted application through its validation, review, and publication lifecycle.
- **Client Creation Schema vs. Worker-Managed Fields:**
  * **Client Creatable Fields ONLY:** `id`, `developerUid`, `submissionSource`, `repoNumericId`, `repoId`, `targetProductId`, `productData`, `status`, `claimStatus`, `createdAt`, `updatedAt`.
  * **Explicitly FORBIDDEN in Client Creation:** `validationResults`, `ownershipProof`, `processingLease`, `adminNotes`, `reviewedBy`, `reviewedAt`, `publishedProductId`. Any client attempt to inject these worker-managed fields violates `keys().hasOnly(...)` and is rejected by Firestore Security Rules.
  * **Worker-Managed Initialization:** Trusted batch workers initialize and update `validationResults`, `ownershipProof`, `processingLease`, `adminNotes`, `reviewedBy`, `reviewedAt`, and `publishedProductId` using the Firebase Admin SDK, which naturally bypasses Security Rules.

```typescript
type SubmissionStatus = 
  | 'queued'               // Awaiting batch worker lease
  | 'processing'           // Actively leased and under validation by GitHub Actions
  | 'needs_review'         // Automated validation passed; awaiting manual admin approval
  | 'approved'             // Qualified for publication; queued for catalog compilation
  | 'catalog_generating'   // Leased by compilation batch
  | 'catalog_committed'    // JSON catalog & manifest committed to Git; awaiting deployment
  | 'deploying'            // GitHub Pages deployment workflow in progress
  | 'deployment_verified'  // Verified via public HTTP manifest and dedicated Pages API
  | 'published'            // Confirmed live on GitHub Pages storefront
  | 'deployment_failed'    // Pages deploy failed/timed out; queued for recompilation retry
  | 'rejected'             // Failed manual review or policy inspection
  | 'abandoned';           // Failed validation or unrecoverable error

type OwnershipProofTier = 
  | 'direct_personal_owner'  // Personal repo matching developer's verified GitHub ID (Auto-Publish eligible)
  | 'org_member'             // Member of repository organization (Requires admin approval)
  | 'committer'              // Commits in Git log but not owner (Requires admin approval)
  | 'community_nominator'    // Third party recommending open-source tool (Requires admin approval)
  | 'unverified';            // Check failed or private repository

interface SubmissionDocument {
  id: string;                        // "sub_issue_{repoId}_{issueNumber}" or "sub_dash_{nanoid}"
  developerUid: string | null;       // Owner Firebase UID (null if unclaimed issue)
  submissionSource: 'dashboard' | 'github_issue';
  repoNumericId: number;             // Immutable numeric GitHub Repo ID e.g. 892341901
  repoId: string;                    // "gh_repo_" + repoNumericId
  targetProductId: string;           // "prod_" + repoNumericId + "_" + productSlug
  claimStatus: 'claimed' | 'unclaimed';
  submitterGithubId?: string;        // Extracted from Issue author for async claiming

  // Developer-Submitted Payload (Enforced by strict whitelist)
  productData: {
    name: string;                    // 3–60 chars
    slug: string;                    // 3–30 chars (lowercase a-z, 0-9, hyphens)
    repoUrl: string;                 // Must match exact "https://github.com/" + repoOwner + "/" + repoName
    repoOwner: string;               // 1-39 chars (valid GitHub handle)
    repoName: string;                // 1-100 chars (valid GitHub repo name)
    releaseTag: string;              // 1-50 chars
    selectedAsset: {
      name: string;                  // 1-128 chars
      extension: string;             // 1-16 chars
      sizeBytes: number;             // Positive integer <= 524,288,000 (500 MiB)
      downloadUrl: string;           // GitHub Release browser download URL (Authoritatively verified against GitHub API)
      assetId: number;               // Numeric asset ID (Authoritatively verified against GitHub API)
    };
    shortDescription: string;        // 10-200 chars
    fullDescription: string;         // 20-10000 chars (Markdown)
    category: string;                // Strict category enum
    iconUrl: string;                 // Optional HTTPS URL <= 500 chars
    screenshots: string[];           // Max 6 HTTPS URLs (each <= 500 chars)
    supportedPlatforms: string[];    // Bounded list: 1-6 unique items subset of ['windows', 'macos', 'linux', 'android', 'ios', 'web']
    version: string;                 // Bounded string in Rules (1-32 chars); Authoritatively validated as SemVer 2.0.0 by worker
    license: string;                 // Bounded string in Rules (1-50 chars); Authoritatively validated against official SPDX catalog by worker
    liveDemoUrl?: string | null;     // Optional HTTPS URL <= 500 chars
  };

  // State Management & Processing Lease (Worker-Managed via Admin SDK)
  status: SubmissionStatus;
  statusReason?: string | null;
  processingLease?: {
    leaseHolder: string | null;
    leaseAcquiredAt: FirebaseFirestore.Timestamp | null;
    leaseExpiresAt: FirebaseFirestore.Timestamp | null;
  } | null;

  // Automated GitHub Verification Results (Worker-Managed via Admin SDK)
  validationResults?: {
    repoExists: boolean;
    repoPublic: boolean;
    repoNotArchived: boolean;
    releaseExists: boolean;
    releaseNotDraft: boolean;
    assetExists: boolean;
    assetSizeMatch: boolean;
    validatedAt: FirebaseFirestore.Timestamp;
  } | null;

  // Repository Ownership Proof (Worker-Managed via Admin SDK)
  ownershipProof?: {
    tier: OwnershipProofTier;
    evidence: string;
    sufficientForAutoPublish: boolean;
    verifiedAt: FirebaseFirestore.Timestamp;
  } | null;

  adminNotes?: string | null;        // Worker-Managed via Admin SDK
  reviewedBy?: string | null;        // Worker-Managed via Admin SDK
  reviewedAt?: FirebaseFirestore.Timestamp | null; // Worker-Managed via Admin SDK
  approvedCatalogRevision?: number | null; // Monotonic catalog revision at which submission was approved (Worker-Managed via Admin SDK; used by Moderation Revision Fence)
  publishedProductId?: string | null; // Worker-Managed via Admin SDK
  publishedGenerationId?: string | null; // Worker-Managed via Admin SDK
  publishedGenerationSequence?: number | null; // Monotonic sequence of generation that published this submission (Worker-Managed via Admin SDK)

  createdAt: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.4 Collection: `claim_requests`
- **Path:** `/claim_requests/{requestId}`
- **Document ID:** Auto-generated ID (`claim_req_{nanoid}`).
- **Description:** Secure queue for claiming unclaimed GitHub Issue Form submissions without giving browser clients write access to submission ownership fields.
- **Client Creation Whitelist:** Clients can ONLY supply `requestId`, `requesterUid`, `targetSubmissionId`, `status`, and `createdAt`.
- **Strictly Blocked Privileged Fields:** `developerUid`, `verifiedGithubId`, `claimStatus`, `isAdmin`, `processedAt`, `reviewedAt`, `reviewedBy`, `resolution`, `rejectionReason`, `workerLease`, and `workerId` are rejected by Security Rules if present in client create requests.

```typescript
interface ClaimRequestDocument {
  id: string;                        // "claim_req_{nanoid}"
  requesterUid: string;              // Must equal request.auth.uid (enforced by Security Rules)
  targetSubmissionId: string;        // Foreign key to submissions/{id}
  status: 'pending' | 'processing' | 'completed' | 'rejected_mismatch' | 'rejected_already_claimed';
  rejectionReason: string | null;    // Worker-managed
  verifiedGithubId: string | null;   // Extracted by worker from Firebase Admin SDK
  processedByWorkerRunId: string | null; // Worker-managed
  createdAt: FirebaseFirestore.Timestamp;
  processedAt: FirebaseFirestore.Timestamp | null; // Worker-managed
}
```

### 2.5 Collection: `moderation_requests` (Action-Bound Proof, Action-Specific Payloads)
- **Path:** `/moderation_requests/{requestId}`
- **Document ID:** Auto-generated ID (`mod_req_{nanoid}`).
- **Description:** Decoupled asynchronous command queue enabling administrators to initiate privileged actions with mandatory fresh GitHub step-up reauthentication.
- **Field Consistency:** The initiating admin UID is stored canonically in `requestedByUid` (matching `request.auth.uid`).
- **ActionHash Classification:** The client-supplied SHA-256 `actionHash` is a **data-integrity checksum** computed over `${action}:${targetDocumentPath}:${actionNonce}:${authTime}`. It detects client-side data tampering in transit, but does **not** serve as a cryptographic signature.
- **Authoritative Security Foundation:**
  1. Authenticated Firebase ID token with fresh `auth_time` (< 300s) and provider `github.com`.
  2. Verified numeric GitHub ID matching (`identities['github.com'][0]`).
  3. Authoritative collaborator permission verification against `Rohitpvt/GitStore` via GitHub REST API (`role_name === 'admin' || role_name === 'maintain'`).
  4. Worker-managed atomic nonce reservation in `/consumed_stepup_nonces`.
  5. Strict worker and security rule validation that `targetDocumentPath` matches the canonical action-to-target mapping and `payload` satisfies action-specific schemas.

#### Action-Specific Payload & Target Binding Matrix

| Action | Target Pattern (`targetDocumentPath`) | Target ID Format (`targetId`) | Permitted Payload Keys | Required Payload Keys | Field Types & Constraints |
|---|---|---|---|---|---|
| `APPROVE_SUBMISSION` | `submissions/${targetId}` | `^[a-zA-Z0-9_-]+$` | `reviewNotes` | None | `reviewNotes`: optional string, <= 1000 chars |
| `REJECT_SUBMISSION` | `submissions/${targetId}` | `^[a-zA-Z0-9_-]+$` | `rejectionReason`, `reviewNotes` | `rejectionReason` | `rejectionReason`: string, 10–1000 chars; `reviewNotes`: optional string, <= 1000 chars |
| `VERIFY_DEVELOPER` | `public_profiles/${targetId}` or `users/${targetId}` | `^[a-zA-Z0-9_-]+$` | `verificationReason`, `evidenceUrl` | `verificationReason` | `verificationReason`: string, 10–500 chars; `evidenceUrl`: optional string, <= 200 chars |
| `REVOKE_DEVELOPER` | `public_profiles/${targetId}` or `users/${targetId}` | `^[a-zA-Z0-9_-]+$` | `revocationReason` | `revocationReason` | `revocationReason`: string, 10–500 chars |
| `DELIST_PRODUCT` | `products/${targetId}` | `^prod_[0-9]+_[a-z0-9-]+$` | `delistReason`, `noticePeriodDays` | `delistReason` | `delistReason`: string, 10–1000 chars; `noticePeriodDays`: optional number, 0–30 |
| `FEATURE_PRODUCT` | `products/${targetId}` | `^prod_[0-9]+_[a-z0-9-]+$` | `featuredRank`, `featureNotes` | `featuredRank` | `featuredRank`: number, 1–100; `featureNotes`: optional string, <= 500 chars |

```typescript
type ModerationAction = 
  | 'APPROVE_SUBMISSION'
  | 'REJECT_SUBMISSION'
  | 'VERIFY_DEVELOPER'
  | 'REVOKE_DEVELOPER'
  | 'DELIST_PRODUCT'
  | 'FEATURE_PRODUCT';

interface StepUpProof {
  providerId: 'github.com';          // Enforced by Security Rules
  authTime: number;                  // Seconds since epoch, strictly matching request.auth.token.auth_time
  actionNonce: string;               // UUIDv4 or 64-char random hex generated on client
  targetDocumentPath: string;        // e.g. "submissions/sub_issue_892341901_42"
  actionHash: string;                // Client checksum: SHA256(`${action}:${targetDocumentPath}:${actionNonce}:${authTime}`)
}

interface ModerationRequestDocument {
  id: string;
  action: ModerationAction;
  targetId: string;                  // Entity identifier e.g. "sub_issue_892341901_42"
  targetDocumentPath: string;        // Full path e.g. "submissions/sub_issue_892341901_42"
  payload: Record<string, any>;      // Action-specific validated schema
  requestedByUid: string;            // Firebase Auth UID (must match request.auth.uid)
  githubUserId: string;              // Immutable numeric GitHub user ID (must match token identities)
  stepUpProof: StepUpProof;
  status: 'pending' | 'processing' | 'completed' | 'rejected' | 'expired';
  rejectionReason: string | null;    // Worker-managed
  processedByWorkerRunId: string | null;
  createdAt: FirebaseFirestore.Timestamp;
  processedAt: FirebaseFirestore.Timestamp | null;
}
```

### 2.6 Collection: `consumed_stepup_nonces` (Worker-Managed Replay Ledger)
- **Path:** `/consumed_stepup_nonces/{nonceHash}`
- **Document ID:** `nonceHash` (SHA-256 hex string of `stepUpProof.actionNonce`).
- **Description:** Internal ledger preventing step-up nonce reuse across different moderation request documents.
- **Client Access:** `allow read, write: if false;` (Internal collection; Admin SDK only).
- **Atomic Reservation Invariant:** In the exact same Firestore transaction that leases a moderation request and transitions it to `processing`, the worker checks whether `/consumed_stepup_nonces/{nonceHash}` exists. If it exists, the transaction fails and the request is rejected with `NONCE_ALREADY_CONSUMED`. If it does not exist, the worker creates the document.

```typescript
interface ConsumedStepUpNonceDocument {
  nonceHash: string;                 // Primary key: SHA256(actionNonce)
  requestId: string;                 // Reference to moderation_requests/{id}
  requestedByUid: string;            // Administrator UID (matches request.requestedByUid)
  action: ModerationAction;          // Moderation action authorized
  targetDocumentPath: string;        // Document modified
  workerRunId: string;               // Actions run ID consuming the nonce
  consumedAt: FirebaseFirestore.Timestamp;
}
```

### 2.7 Collection: `products`
- **Path:** `/products/{productId}`
- **Document ID:** Deterministic Globally Unique Key: `prod_{repoNumericId}_{productSlug}` (e.g. `prod_892341901_viewer`).
- **Description:** Authoritative live catalog of active software products. Supports multiple products per repository without cross-repository collisions.

```typescript
interface ProductDocument {
  id: string;                        // "prod_" + repoNumericId + "_" + productSlug
  slug: string;                      // Unique slug within the repository (e.g., "viewer")
  name: string;                      // Application name
  repoNumericId: number;             // GitHub immutable repo ID e.g. 892341901
  repoId: string;                    // "gh_repo_" + repoNumericId
  developerUid: string | null;       // Owner UID (null if unclaimed)
  developerSnapshot: {
    displayName: string;
    avatarUrl: string;
    githubUsername: string;
    isVerifiedDeveloper: boolean;    // Trust badge
  };

  // Upstream Repository Metadata (Updated on repo rename/transfer)
  repoUrl: string;
  repoOwner: string;
  repoName: string;
  releaseTag: string;
  downloadAsset: {
    name: string;
    extension: string;
    sizeBytes: number;
    downloadUrl: string;
    assetId: number;
    releasePublishedAt: FirebaseFirestore.Timestamp;
  };

  shortDescription: string;
  fullDescription: string;
  category: string;
  iconUrl: string;
  screenshots: string[];
  supportedPlatforms: string[];
  version: string;
  license: string;
  liveDemoUrl: string | null;

  metrics: {
    githubStars: number;
    githubForks: number;
    releaseDownloads: number;
    gitStoreViews: number;
  };

  catalogGenerationId: string;       // Matched with deployed data/deployment-manifest.json
  catalogCommitSha: string;          // Git commit SHA that published this product
  catalogDigest: string;             // SHA-256 digest of catalog state (sha256:v1:...)
  status: 'active' | 'delisted' | 'flagged';
  featured: boolean;
  publishedAt: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.8 Collection: `system_deployments` (Internal Tracking)
- **Path:** `/system_deployments/{generationId}`
- **Document ID:** `generationId` (e.g., `gen_1730635200000_a1b2c3d4`).
- **Description:** Tracks compilation generations, commit provenance, staged digest validation, deployment verification, and independently recoverable submission fan-out.
- **Client Access:** `allow read, write: if false;` (Internal collection; Admin SDK only).

```typescript
interface SystemDeploymentDocument {
  generationId: string;              // Primary key
  generationSequence: number;        // Strictly monotonic generation counter
  capturedRevision: number;          // Mandatory catalog revision snapshot (never substituted with sequence)
  finalizedRevision?: number | null; // Revision confirmed at publication finalization
  catalogCommitSha: string | null;   // 40-char git commit SHA on origin/main
  catalogDigest: string;             // Expected canonical digest (sha256:v1:...)
  stagedDigest?: string | null;      // Digest recomputed from staged site/data files
  pagesDeploymentReference: string | number | null; // Numeric ID or commit SHA reference
  status: 'pending' | 'catalog_committed' | 'deploying' | 'published' | 'finalized' | 'failed' | 'stale_aborted' | 'deployment_retained';
  reconciliationStatus: 'none' | 'sha_recovered' | 'pages_dispatched' | 'pending_retry' | 'fanout_pending' | 'reconciled' | 'manual_review' | 'abandoned';
  lastObservedPagesStatus: 'succeed' | 'queued' | 'in_progress' | 'errored' | null;

  // Bounded & Independently Recoverable Submission Fan-Out Summary State (O(1) Document Size)
  fanoutPreparationStatus: 'pending' | 'preparing' | 'prepared'; // Atomic preparation lock state
  preparationOwner: string | null;   // Unique worker execution identifier holding preparation lock (or null)
  preparationExpiresAt: FirebaseFirestore.Timestamp | null; // Atomic 15-minute preparation lock expiry timestamp
  workPlanDigest: string | null;     // Mandatory canonical 64-char hex sha256("workplan:v1:" + generationId + ":" + sortedUniqueIds.join(','))
  submissionFanoutStatus: 'pending' | 'in_progress' | 'completed' | 'partial' | 'failed' | 'manual_review';
  expectedSubmissionCount: number;   // Total expected submissions across all batches (O(1) scalar)
  completedSubmissionCount: number;  // Confirmed successfully processed/skipped submissions (O(1) scalar)
  totalFanoutBatches: number;        // Total deterministic batch documents in subcollection (O(1) scalar)
  completedFanoutBatches: number;    // Number of completed batches in subcollection (O(1) scalar)
  fanoutAttempts: number;            // Number of fan-out execution attempts across the generation
  lastFanoutAttemptAt?: FirebaseFirestore.Timestamp | null;
  lastFanoutError?: string | null;
  manualReviewAt?: FirebaseFirestore.Timestamp | null; // Timestamp when manual review was triggered

  attemptCount: number;
  createdAt: FirebaseFirestore.Timestamp;
  publishedAt?: FirebaseFirestore.Timestamp | null;
  finalizedAt?: FirebaseFirestore.Timestamp | null;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.11 Subcollection: `fanout_batches` (Bounded Submission Fan-Out Storage)
- **Path:** `/system_deployments/{generationId}/fanout_batches/{batchId}`
- **Document ID:** Deterministic zero-padded batch ID (`batch_00000`, `batch_00001`, ..., `batch_NNNNN`).
- **Description:** Stores partitioned chunks of expected submission IDs (maximum 400 IDs per document, occupying ~55 KB at max 128 chars/ID, strictly ~5.3% of Firestore's 1 MiB limit). Replaces unbounded arrays on the parent generation document with bounded, quota-aware child documents.
- **Client Access:** `allow read, write: if false;` (Internal subcollection; Admin SDK only).

```typescript
interface FanoutBatchDocument {
  generationId: string;              // Parent generation ID
  batchIndex: number;                // 0-indexed sequential batch counter
  batchId: string;                   // Deterministic ID (e.g., "batch_00000")
  submissionIds: string[];           // Bounded array: max 400 /submissions/{id} document IDs (conservative size: 400 * 128 B = 51.2 KB + JSON overhead ~55 KB << 1 MiB limit)
  batchDigest: string;               // Canonical sha256("batch:v1:" + generationId + ":" + batchIndex + ":" + chunkIds.join(','))
  status: 'pending' | 'processing' | 'completed' | 'manual_review';
  expectedCount: number;             // submissionIds.length (<= 400)
  completedCount: number;            // Confirmed updated or idempotently verified submissions
  skippedCount: number;              // Submissions skipped due to newer generation sequence, revision fence, or product delisting
  fanoutAttempts: number;            // Number of execution attempts for this specific batch
  leaseOwner: string | null;         // Unique worker execution identifier holding active processing lease (or null)
  leaseAcquiredAt?: FirebaseFirestore.Timestamp | null; // Timestamp when active lease was acquired
  leaseExpiresAt: FirebaseFirestore.Timestamp | null; // Atomic 15-minute lease expiry timestamp
  lastHeartbeatAt?: FirebaseFirestore.Timestamp | null; // Timestamp when lease heartbeat was last renewed
  lastAttemptAt?: FirebaseFirestore.Timestamp | null;
  lastError?: string | null;
  failedAt?: FirebaseFirestore.Timestamp | null;
  createdAt: FirebaseFirestore.Timestamp;
  completedAt?: FirebaseFirestore.Timestamp | null;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

---

## 3. Provenance-Preserving Staging & Error-Classified Reconciler

### 3.1 Single Authoritative Staging Procedure
To prevent artifact corruption and ensure exact provenance:
1. Identify the exact target `catalogCommitSha`.
2. Completely remove any existing staging directory: `rm -rf site/`.
3. Construct `site/` exclusively from the target commit (using git archive or dedicated clean checkout).
4. Place static pages (`index.html`, `admin.html`, `dashboard.html`, `product.html`, `explore.html`, `submit.html`), `assets/`, and compiled `data/` into `site/`.
5. Recompute the catalog digest directly from the staged files: `stagedDigest = computeCatalogDigest('site/data')`.
6. Assert `stagedDigest === gen.catalogDigest`. If mismatch, delete `site/` and abort staging immediately.
7. Confirm `site/data/deployment-manifest.json` matches `{ generationId, generationSequence, catalogDigest }`.
8. Do not copy from `main` or workspace files after staging completes.
9. Preserve `site/` intact until `actions/upload-pages-artifact@v5.0.0` packages it.

### 3.2 Output Bridge Implementation (`scripts/processor/output-bridge.js`)
A Node.js function returning an object does not populate Actions step outputs. The output bridge writes directly to `$GITHUB_OUTPUT`:

```javascript
import fs from 'node:fs';

export function emitStepOutput(name, value) {
  const outputPath = process.env.GITHUB_OUTPUT;
  if (!outputPath) {
    console.warn(`[WARN] GITHUB_OUTPUT not set. Step output '${name}' not exported.`);
    return;
  }
  const strValue = String(value);
  if (strValue.includes('\n') || strValue.includes('\r')) {
    throw new Error(`SECURITY_ERROR: Step output '${name}' contains newline characters`);
  }
  fs.appendFileSync(outputPath, `${name}=${strValue}\n`, 'utf8');
}
```

### 3.3 Executable Reconciler Logic with Error Classification & 2-Tier Finalization

```javascript
import fs from 'node:fs';
import path from 'node:path';
import crypto from 'node:crypto';
import { execSync } from 'node:child_process';
import { emitStepOutput } from './output-bridge.js';

export function evaluateGenerationOrdering(artifactSeq, artifactGenId, activeSeq, activeGenId) {
  if (artifactSeq < activeSeq) return 'REJECT_STALE';
  if (artifactSeq === activeSeq && artifactGenId === activeGenId) return 'ALLOW_IDEMPOTENT';
  if (artifactSeq === activeSeq && artifactGenId !== activeGenId) return 'HALT_CONFLICT';
  return 'ALLOW_NEW';
}

/**
 * Strict error classifier for GitHub Commits & Contents API inspection during commit recovery.
 * Invariant: Only a confirmed HTTP 404 for missing manifest blob is treated as a benign skip.
 * All other failures are classified as incomplete, retryable, corrupt, or fatal.
 */
export function classifyCommitInspectionError(err) {
  // 1. Benign absence: Only a confirmed 404 indicating manifest blob does not exist in that commit
  if (err.status === 404) {
    return { classification: 'CANDIDATE_NO_MANIFEST', isFatal: false, isIncomplete: false, isCorrupt: false, retryable: false };
  }

  // 2. Fatal auth failure (bad or expired token)
  if (err.status === 401) {
    return { classification: 'API_AUTH_ERROR', isFatal: true, isIncomplete: true, isCorrupt: false, retryable: false };
  }

  // 3. Permission or secondary rate limit / abuse detection
  if (err.status === 403) {
    return { classification: 'API_PERMISSION_OR_RATE_LIMIT', isFatal: false, isIncomplete: true, isCorrupt: false, retryable: true };
  }

  // 4. Rate limit (primary API rate limit hit)
  if (err.status === 429) {
    return { classification: 'API_RATE_LIMIT', isFatal: false, isIncomplete: true, isCorrupt: false, retryable: true };
  }

  // 5. Upstream server error (5xx)
  if (err.status >= 500 && err.status < 600) {
    return { classification: 'API_SERVER_ERROR', isFatal: false, isIncomplete: true, isCorrupt: false, retryable: true };
  }

  // 6. Network timeouts, connection resets, DNS failures
  if (err.code === 'ETIMEDOUT' || err.code === 'ECONNRESET' || err.code === 'ENOTFOUND' || err.name === 'AbortError' || (err.message && /timeout|reset|network|fetch/i.test(err.message))) {
    return { classification: 'NETWORK_ERROR', isFatal: false, isIncomplete: true, isCorrupt: false, retryable: true };
  }

  // 7. Malformed JSON / SyntaxError during manifest blob parsing
  if (err instanceof SyntaxError || (err.message && /JSON/i.test(err.message))) {
    return { classification: 'CANDIDATE_CORRUPT_MANIFEST', isFatal: false, isIncomplete: false, isCorrupt: true, retryable: false };
  }

  // 8. Missing response content (malformed payload from Contents API)
  if (err.message && /content/i.test(err.message)) {
    return { classification: 'API_MALFORMED_RESPONSE', isFatal: false, isIncomplete: true, isCorrupt: true, retryable: true };
  }

  // 9. All other unexpected errors
  return { classification: 'UNEXPECTED_ERROR', isFatal: false, isIncomplete: true, isCorrupt: false, retryable: true };
}

/**
 * Reconciles live deployed GitHub Pages state, recovers unpersisted Git commit SHAs,
 * classifies commit inspection errors, and executes 2-tier finalization.
 */
async function reconcilePublishedState(db, octokit, pagesUrl) {
  let handoffResult = { deployNeeded: false };

  // 1. Fetch current global deployment and catalog state
  const stateDoc = await db.collection('system_config').doc('deployment_state').get();
  const state = stateDoc.exists ? stateDoc.data() : { latestPublishedSequence: 0, latestPublishedGenerationId: null };

  const catalogStateDoc = await db.collection('system_config').doc('catalog_state').get();
  const catalogState = catalogStateDoc.exists ? catalogStateDoc.data() : { latestRequestedRevision: 0, latestPublishedRevision: 0 };

  // 2. Fetch public deployment manifest from live GitHub Pages CDN
  let liveManifest = null;
  try {
    const res = await fetch(`${pagesUrl}/data/deployment-manifest.json?t=${Date.now()}`, {
      headers: { 'Cache-Control': 'no-cache' }
    });
    if (res.ok) liveManifest = await res.json();
  } catch (err) {
    console.warn('[Reconciler] Could not reach public Pages manifest.', err.message);
  }

  // 3. Shared Generation Ordering Evaluation
  if (liveManifest) {
    const orderDecision = evaluateGenerationOrdering(
      liveManifest.generationSequence,
      liveManifest.generationId,
      state.latestPublishedSequence,
      state.latestPublishedGenerationId
    );

    if (orderDecision === 'REJECT_STALE') {
      console.warn(`[Reconciler] Case 1 (Stale): CDN manifest sequence (${liveManifest.generationSequence}) < latest published (${state.latestPublishedSequence}). Aborting.`);
      return handoffResult;
    } else if (orderDecision === 'ALLOW_IDEMPOTENT') {
      console.log(`[Reconciler] Case 2 (Idempotent): CDN matches currently published generation ${liveManifest.generationId}.`);
    } else if (orderDecision === 'HALT_CONFLICT') {
      console.error(`[Reconciler] CRITICAL SECURITY ALERT: Case 3 (Sequence Conflict). Manifest seq ${liveManifest.generationSequence} claimed by ${liveManifest.generationId}, but Firestore has ${state.latestPublishedGenerationId}. Halting.`);
      return handoffResult;
    }
  }

  // 4. Query active deployments pending finalization or retry across non-terminal reconciliation states
  const pendingGenerations = await db.collection('system_deployments')
    .where('status', 'in', ['pending', 'built', 'dispatched', 'deploying', 'catalog_committed', 'deployment_retained'])
    .where('reconciliationStatus', 'in', ['none', 'sha_recovered', 'pages_dispatched', 'pending_retry'])
    .get();

  for (const genDoc of pendingGenerations.docs) {
    const gen = genDoc.data();
    console.log(`[Reconciler] Evaluating generation ${gen.generationId} (seq ${gen.generationSequence}) for recovery...`);

    const orderDecision = evaluateGenerationOrdering(
      gen.generationSequence,
      gen.generationId,
      state.latestPublishedSequence,
      state.latestPublishedGenerationId
    );

    if (orderDecision === 'REJECT_STALE') {
      console.warn(`[Reconciler] Retiring superseded generation ${gen.generationId} (seq ${gen.generationSequence} < ${state.latestPublishedSequence}). Marking stale_aborted.`);
      await genDoc.ref.update({
        status: 'stale_aborted',
        reconciliationStatus: 'stale_aborted',
        updatedAt: new Date()
      });
      continue;
    }

    if (orderDecision === 'HALT_CONFLICT') {
      console.error(`[Reconciler] CRITICAL HALT: Sequence collision in pending generation ${gen.generationId} at sequence ${gen.generationSequence}. Routing to manual_review.`);
      await genDoc.ref.update({
        status: 'failed',
        reconciliationStatus: 'manual_review',
        updatedAt: new Date()
      });
      continue;
    }

    let isPagesSucceed = false;
    let isManifestVerified = false;

    if (gen.pagesDeploymentReference) {
      try {
        const pagesRes = await octokit.rest.repos.getPagesDeployment({
          owner: 'Rohitpvt',
          repo: 'GitStore',
          pages_deployment_id: gen.pagesDeploymentReference
        });
        if (pagesRes.data && pagesRes.data.status === 'succeed') {
          isPagesSucceed = true;
          await genDoc.ref.update({ lastObservedPagesStatus: 'succeed', updatedAt: new Date() });
        }
      } catch (pagesErr) {
        console.warn(`[Reconciler] Pages deployment inspection error for ${gen.generationId}: ${pagesErr.message}`);
      }
    }

    if (isPagesSucceed && liveManifest) {
      if (liveManifest.generationId === gen.generationId && liveManifest.catalogDigest === gen.catalogDigest) {
        isManifestVerified = true;
      }
    }

    // Two-Tier Transactional Finalization
    if (isPagesSucceed && isManifestVerified) {
      console.log(`[Reconciler] Healing generation ${gen.generationId}. Finalizing Firestore.`);

      try {
        // TIER 1: Atomic Global State Anchor Transaction (3 writes)
        await db.runTransaction(async (transaction) => {
          // READ 1: catalog_state
          const catalogStateRef = db.collection('system_config').doc('catalog_state');
          const liveCatDoc = await transaction.get(catalogStateRef);
          const liveCat = liveCatDoc.exists ? liveCatDoc.data() : { latestRequestedRevision: 0, latestPublishedRevision: 0 };

          // READ 2: deployment_state
          const deployStateRef = db.collection('system_config').doc('deployment_state');
          const liveDepDoc = await transaction.get(deployStateRef);
          const liveDep = liveDepDoc.exists ? liveDepDoc.data() : { latestPublishedSequence: 0, latestPublishedGenerationId: null };

          // READ 3: generation record
          const genRef = db.collection('system_deployments').doc(gen.generationId);
          const genFreshDoc = await transaction.get(genRef);
          if (!genFreshDoc.exists) throw new Error('GENERATION_NOT_FOUND');
          const freshGen = genFreshDoc.data();

          if (typeof freshGen.capturedRevision !== 'number') {
            throw new Error(`INVALID_CAPTURED_REVISION: Not a number in ${gen.generationId}`);
          }

          // Complete shared ordering decision
          const orderDecision = evaluateGenerationOrdering(
            freshGen.generationSequence,
            freshGen.generationId,
            liveDep.latestPublishedSequence,
            liveDep.latestPublishedGenerationId
          );

          if (orderDecision === 'REJECT_STALE') {
            throw new Error(`STALE_GENERATION_ABORT: sequence ${freshGen.generationSequence} < latestPublishedSequence ${liveDep.latestPublishedSequence}`);
          }
          if (orderDecision === 'HALT_CONFLICT') {
            throw new Error(`HALT_CONFLICT: sequence collision at ${freshGen.generationSequence}`);
          }

          // Revision monotonicity check
          if (freshGen.capturedRevision < liveCat.latestPublishedRevision) {
            throw new Error(`STALE_REVISION_ABORT: capturedRevision ${freshGen.capturedRevision} < latestPublishedRevision ${liveCat.latestPublishedRevision}`);
          }
          if (freshGen.capturedRevision > liveCat.latestRequestedRevision) {
            throw new Error(`FUTURE_REVISION_ANOMALY: capturedRevision ${freshGen.capturedRevision} > latestRequestedRevision ${liveCat.latestRequestedRevision}`);
          }

          // Invariant: Fan-out work plan must be fully prepared and verified before Tier 1 publication commits!
          if (freshGen.fanoutPreparationStatus !== 'prepared') {
            throw new Error(`PREPARATION_INCOMPLETE_ABORT: Generation ${freshGen.generationId} fan-out batches must be prepared and verified before Tier 1 publication (current status: ${freshGen.fanoutPreparationStatus || 'pending'}).`);
          }

          // Invariant: Prepared generation must have a valid canonical 64-char hex workPlanDigest
          if (!freshGen.workPlanDigest || typeof freshGen.workPlanDigest !== 'string' || !/^[0-9a-f]{64}$/.test(freshGen.workPlanDigest)) {
            throw new Error(`PREPARED_DIGEST_MISSING_OR_INVALID_ABORT: Generation ${freshGen.generationId} marked prepared but lacks a valid 64-character hex workPlanDigest.`);
          }

          const nextPublishedRev = Math.max(liveCat.latestPublishedRevision, freshGen.capturedRevision);
          const isDirty = liveCat.latestRequestedRevision > nextPublishedRev;

          // WRITE 1: Advance deployment_state
          transaction.set(deployStateRef, {
            latestPublishedSequence: freshGen.generationSequence,
            latestPublishedGenerationId: freshGen.generationId,
            latestPublishedCommitSha: freshGen.catalogCommitSha,
            latestPublishedCatalogDigest: freshGen.catalogDigest,
            status: 'idle',
            lastPublishedAt: new Date(),
            updatedAt: new Date()
          }, { merge: true });

          // WRITE 2: Advance catalog_state
          transaction.set(catalogStateRef, {
            latestPublishedRevision: nextPublishedRev,
            isDirty: isDirty,
            lastPublishedGenerationId: freshGen.generationId,
            lastPublishedAt: new Date(),
            updatedAt: new Date()
          }, { merge: true });

          // WRITE 3: Advance system_deployments record to published (with fan-out pending)
          transaction.update(genRef, {
            status: 'published',
            reconciliationStatus: 'fanout_pending',
            submissionFanoutStatus: 'pending',
            completedSubmissionCount: 0,
            completedFanoutBatches: 0,
            fanoutAttempts: 0,
            finalizedRevision: nextPublishedRev,
            verifiedAt: new Date(),
            updatedAt: new Date()
          });
        });

        // TIER 2: Execute Independently Recoverable Submission Status Fan-Out
        await executeSubmissionFanout(db, genDoc.ref, gen);

      } catch (err) {
        if (err.message && (err.message.includes('STALE_GENERATION_ABORT') || err.message.includes('STALE_REVISION_ABORT'))) {
          console.warn(`[Reconciler] Stale finalization aborted safely. Marking generation ${gen.generationId} as stale_aborted.`);
          await genDoc.ref.update({ status: 'stale_aborted', reconciliationStatus: 'stale_aborted', updatedAt: new Date() });
        } else if (err.message && err.message.includes('PREPARATION_INCOMPLETE_ABORT')) {
          console.warn(`[Reconciler] Fan-out preparation incomplete for generation ${gen.generationId}. Retaining generation in pending.`);
        } else if (err.message && err.message.includes('PREPARED_DIGEST_MISSING_OR_INVALID_ABORT')) {
          console.error(`[Reconciler] Prepared digest missing or invalid for generation ${gen.generationId}. Routing to manual_review.`);
          await genDoc.ref.update({ status: 'failed', reconciliationStatus: 'manual_review', updatedAt: new Date() });
        } else {
          throw err;
        }
      }
    }
  }

  // 5. Query published generations with incomplete submission fan-out via stable immutable cursor pagination
  // Cursor Ordering Invariant: Ordering is strictly on immutable 'generationSequence' ASC.
  // Because 'generationSequence' is immutable, status transitions ('pending' -> 'in_progress' -> 'completed')
  // during traversal do NOT shift document cursor positions or cause skipped records.
  const GEN_PAGE_SIZE = 10;
  let lastGenDoc = null;
  let hasMoreGenerations = true;

  while (hasMoreGenerations) {
    let genQuery = db.collection('system_deployments')
      .where('status', '==', 'published')
      .where('submissionFanoutStatus', 'in', ['pending', 'in_progress', 'partial'])
      .orderBy('generationSequence', 'asc')
      .limit(GEN_PAGE_SIZE);

    if (lastGenDoc) {
      genQuery = genQuery.startAfter(lastGenDoc);
    }

    const genPageSnap = await genQuery.get();

    if (genPageSnap.empty) {
      hasMoreGenerations = false;
      break;
    }

    for (const fanoutDoc of genPageSnap.docs) {
      const fanoutGen = fanoutDoc.data();
      console.log(`[Reconciler] Resuming incomplete submission fan-out for generation ${fanoutGen.generationId} (${fanoutGen.completedFanoutBatches || 0}/${fanoutGen.totalFanoutBatches || 0} batches completed).`);
      await executeSubmissionFanout(db, fanoutDoc.ref, fanoutGen);
    }

    if (genPageSnap.size < GEN_PAGE_SIZE) {
      hasMoreGenerations = false;
    } else {
      lastGenDoc = genPageSnap.docs[genPageSnap.docs.length - 1];
    }
  }

  return handoffResult;
}

/**
 * Helper: Atomically claims the preparation lock for a generation's work plan.
 * Prevents concurrent workers from preparing different work plans for the same generation.
 */
async function acquirePreparationLock(db, genDocRef, workerExecutionId, workPlanDigest) {
  return await db.runTransaction(async (tx) => {
    const snap = await tx.get(genDocRef);
    if (!snap.exists) {
      throw new Error(`GENERATION_NOT_FOUND: Generation document ${genDocRef.id} does not exist`);
    }
    const freshGen = snap.data();
    const now = Date.now();

    // 1. If already prepared, validate digest format and check for immutability conflicts
    if (freshGen.fanoutPreparationStatus === 'prepared') {
      if (!freshGen.workPlanDigest || typeof freshGen.workPlanDigest !== 'string' || !/^[0-9a-f]{64}$/.test(freshGen.workPlanDigest)) {
        throw new Error(`PREPARED_DIGEST_MISSING_OR_INVALID_ABORT: Generation ${genDocRef.id} marked prepared but lacks a valid 64-character hex workPlanDigest`);
      }
      if (freshGen.workPlanDigest !== workPlanDigest) {
        throw new Error(`WORKPLAN_MUTATION_CONFLICT: Generation ${genDocRef.id} is already marked 'prepared' with digest ${freshGen.workPlanDigest}, but received differing work-plan digest ${workPlanDigest}. Mutation forbidden.`);
      }
      return { acquired: false, status: 'already_prepared', genData: freshGen };
    }

    // 2. If currently being prepared by another worker, check lock expiration
    if (freshGen.fanoutPreparationStatus === 'preparing') {
      const lockExpiresAtMillis = freshGen.preparationExpiresAt ? freshGen.preparationExpiresAt.toMillis() : 0;
      if (lockExpiresAtMillis > now && freshGen.preparationOwner !== workerExecutionId) {
        return {
          acquired: false,
          status: 'locked',
          owner: freshGen.preparationOwner,
          expiresAt: freshGen.preparationExpiresAt
        };
      }
      // Lock expired or held by same worker: safe to reclaim/renew
    }

    // 3. Claim atomic 15-minute preparation ownership lock
    const lockExpiry = new Date(now + 15 * 60 * 1000);
    tx.update(genDocRef, {
      fanoutPreparationStatus: 'preparing',
      preparationOwner: workerExecutionId,
      preparationExpiresAt: lockExpiry,
      updatedAt: new Date(now)
    });

    return { acquired: true, status: 'acquired', genData: freshGen };
  });
}

/**
 * Helper: Bounded cursor paginated readback verification across all child batch documents.
 * Verifies that every expected batch document exists, matches batchIndex, expectedCount,
 * canonical batchDigest, and element-by-element submission IDs.
 */
async function verifyPersistedBatches(genDocRef, genId, uniqueIds, totalBatches) {
  const BATCH_SIZE = 400;
  const batchesCol = genDocRef.collection('fanout_batches');
  const VERIFY_PAGE_SIZE = 50;
  let verifiedCount = 0;
  let lastVerifyDoc = null;
  let hasMoreVerify = true;

  while (hasMoreVerify) {
    let q = batchesCol.orderBy('batchIndex', 'asc').limit(VERIFY_PAGE_SIZE);
    if (lastVerifyDoc) {
      q = q.startAfter(lastVerifyDoc);
    }
    const snap = await q.get();

    if (snap.empty) {
      hasMoreVerify = false;
      break;
    }

    for (const doc of snap.docs) {
      const bData = doc.data();
      const bIdx = bData.batchIndex;

      if (typeof bIdx !== 'number' || bIdx < 0 || bIdx >= totalBatches) {
        throw new Error(`FANOUT_PREPARATION_VERIFICATION_FAILED: Batch ${doc.id} has invalid or out-of-bounds batchIndex ${bIdx} (totalBatches: ${totalBatches})`);
      }

      const expectedChunk = uniqueIds.slice(bIdx * BATCH_SIZE, (bIdx + 1) * BATCH_SIZE);
      const expectedBatchDigest = crypto.createHash('sha256')
        .update(`batch:v1:${genId}:${bIdx}:${expectedChunk.join(',')}`)
        .digest('hex');

      if (bData.generationId !== genId) {
        throw new Error(`FANOUT_PREPARATION_VERIFICATION_FAILED: Batch ${doc.id} generationId mismatch (${bData.generationId} != ${genId})`);
      }
      if (bData.batchId !== doc.id) {
        throw new Error(`FANOUT_PREPARATION_VERIFICATION_FAILED: Batch ${doc.id} batchId field mismatch (${bData.batchId} != ${doc.id})`);
      }
      if (bData.expectedCount !== expectedChunk.length) {
        throw new Error(`FANOUT_PREPARATION_VERIFICATION_FAILED: Batch ${doc.id} expectedCount mismatch (${bData.expectedCount} != ${expectedChunk.length})`);
      }
      if (bData.batchDigest !== expectedBatchDigest) {
        throw new Error(`FANOUT_PREPARATION_VERIFICATION_FAILED: Batch ${doc.id} batchDigest mismatch (${bData.batchDigest} != ${expectedBatchDigest})`);
      }
      if (!Array.isArray(bData.submissionIds) || bData.submissionIds.length !== expectedChunk.length) {
        throw new Error(`FANOUT_PREPARATION_VERIFICATION_FAILED: Batch ${doc.id} submissionIds length mismatch (${bData.submissionIds?.length} != ${expectedChunk.length})`);
      }
      for (let k = 0; k < expectedChunk.length; k++) {
        if (bData.submissionIds[k] !== expectedChunk[k]) {
          throw new Error(`FANOUT_PREPARATION_VERIFICATION_FAILED: Batch ${doc.id} submissionId mismatch at position ${k} (${bData.submissionIds[k]} != ${expectedChunk[k]})`);
        }
      }

      verifiedCount++;
    }

    if (snap.size < VERIFY_PAGE_SIZE) {
      hasMoreVerify = false;
    } else {
      lastVerifyDoc = snap.docs[snap.docs.length - 1];
    }
  }

  if (verifiedCount !== totalBatches) {
    throw new Error(`FANOUT_PREPARATION_VERIFICATION_FAILED: Verified batch count ${verifiedCount} != expected totalBatches ${totalBatches}`);
  }
}

/**
 * Deterministically prepares bounded fan-out child batch documents for a generation's work plan.
 * Must execute and verify complete batch creation BEFORE Tier 1 publication commits.
 *
 * Invariants Enforced (Phase 0.1.13):
 * 1. Fresh Document Read: Never relies on stale genData passed into the function.
 * 2. Concurrency-Safe Preparation Lock: Acquires an atomic 15-minute ownership lock on the parent generation doc.
 * 3. Fail-Closed Input Validation: Validates EVERY input submission ID format (/^sub_[a-zA-Z0-9_-]+$/, 8-128 chars).
 *    Never silently filters or drops malformed IDs; throws INVALID_SUBMISSION_ID_ABORT.
 * 4. Deterministic Deduplication & Sorting: Converts IDs via Set and sorts deterministically.
 * 5. Mandatory Canonical Work-Plan Digest: Computes 64-char hex sha256("workplan:v1:" + genId + ":" + sortedUniqueIds.join(',')).
 * 6. Deep Verification on Idempotent Retry: If already marked prepared, verifies persisted child batches before returning.
 * 7. Active Batch Content Protection: Never modifies submissionIds or batchDigest on actively leased batches.
 *    If active batch content differs from expected plan, fails closed with ACTIVE_BATCH_MUTATION_CONFLICT.
 * 8. Completed Batch Immutability: Completed batches remain strictly immutable; digest mismatch throws COMPLETED_BATCH_CORRUPT_CONFLICT.
 * 9. Bounded Cleanup Pagination: Queries stale batches in bounded pages (limit 50). Never deletes active or completed batches.
 * 10. Bounded Readback Verification: Deep readback verification is paginated in chunks of 50.
 * 11. Atomic Final Preparation Marker: Only persists fanoutPreparationStatus: 'prepared' after deep readback succeeds.
 */
export async function prepareFanoutBatches(db, genDocRef, rawSubmissionIds, workerExecutionId) {
  const genId = genDocRef.id;

  // 1. Fail-Closed Input Validation: Validate every input ID, never silently discard
  if (!Array.isArray(rawSubmissionIds)) {
    throw new Error(`INVALID_SUBMISSION_IDS_ARRAY: rawSubmissionIds must be an array for generation ${genId}`);
  }

  for (let idx = 0; idx < rawSubmissionIds.length; idx++) {
    const id = rawSubmissionIds[idx];
    if (typeof id !== 'string' || id.length < 8 || id.length > 128 || !/^sub_[a-zA-Z0-9_-]+$/.test(id)) {
      throw new Error(`INVALID_SUBMISSION_ID_ABORT: Malformed submission ID at index ${idx}: '${String(id)}'`);
    }
  }

  // 2. Deterministic Deduplication and Lexicographical Sorting
  const uniqueIds = Array.from(new Set(rawSubmissionIds)).sort();

  // 3. Canonical Work-Plan Digest & Format Validation
  const workPlanDigest = crypto.createHash('sha256')
    .update(`workplan:v1:${genId}:${uniqueIds.join(',')}`)
    .digest('hex');

  if (!/^[0-9a-f]{64}$/.test(workPlanDigest)) {
    throw new Error(`INVALID_WORKPLAN_DIGEST_FORMAT: Computed digest '${workPlanDigest}' does not match 64-char hex format`);
  }

  const BATCH_SIZE = 400;
  const totalBatches = Math.ceil(uniqueIds.length / BATCH_SIZE) || (uniqueIds.length === 0 ? 0 : 1);
  const batchesCol = genDocRef.collection('fanout_batches');

  // 4. Atomic Preparation Lock Acquisition (prevents concurrent conflicting preparations)
  const lockResult = await acquirePreparationLock(db, genDocRef, workerExecutionId, workPlanDigest);

  if (!lockResult.acquired) {
    if (lockResult.status === 'already_prepared') {
      console.log(`[Fanout Preparation] Generation ${genId} already prepared with verified digest ${workPlanDigest}. Verifying child batches on retry...`);
      // Invariant: Verify persisted batch plan even on an idempotent retry
      await verifyPersistedBatches(genDocRef, genId, uniqueIds, totalBatches);
      console.log(`[Fanout Preparation] Idempotent readback verification passed for generation ${genId}.`);
      return;
    }

    if (lockResult.status === 'locked') {
      throw new Error(`PREPARATION_LOCKED_BY_OTHER_WORKER: Generation ${genId} fan-out preparation is actively locked by worker '${lockResult.owner}' until ${lockResult.expiresAt.toDate().toISOString()}. Concurrency conflict.`);
    }
  }

  // 5. Bounded Stale Batch Cleanup from prior aborted preparations
  const CLEANUP_PAGE_SIZE = 50;
  let lastCleanupDoc = null;
  let hasMoreCleanup = true;

  while (hasMoreCleanup) {
    let q = batchesCol.orderBy('batchIndex', 'asc').limit(CLEUP_PAGE_SIZE = 50);
    if (lastCleanupDoc) {
      q = q.startAfter(lastCleanupDoc);
    }
    const snap = await q.get();

    if (snap.empty) {
      hasMoreCleanup = false;
      break;
    }

    for (const existingDoc of snap.docs) {
      const existingData = existingDoc.data();
      if (typeof existingData.batchIndex !== 'number' || existingData.batchIndex >= totalBatches || !existingDoc.id.startsWith('batch_')) {
        // Invariant: Never delete active or completed batches during stale cleanup
        const isCompleted = existingData.status === 'completed';
        const now = Date.now();
        const isActiveLease = existingData.status === 'processing' && existingData.leaseExpiresAt && existingData.leaseExpiresAt.toMillis() > now;

        if (isCompleted || isActiveLease) {
          console.warn(`[Fanout Preparation] Stale batch ${existingDoc.id} (index ${existingData.batchIndex}) is ${isCompleted ? 'completed' : 'actively leased'}. Retaining without deletion.`);
          continue;
        }

        console.warn(`[Fanout Preparation] Deleting unleased stale batch doc ${existingDoc.id} (index ${existingData.batchIndex}) from generation ${genId}.`);
        await existingDoc.ref.delete();
      }
    }

    if (snap.size < 50) {
      hasMoreCleanup = false;
    } else {
      lastCleanupDoc = snap.docs[snap.docs.length - 1];
    }
  }

  // 6. Persist deterministic batch documents with batchDigest & Active Batch Protection
  for (let bIndex = 0; bIndex < totalBatches; bIndex++) {
    const chunkIds = uniqueIds.slice(bIndex * BATCH_SIZE, (bIndex + 1) * BATCH_SIZE);
    const batchId = `batch_${String(bIndex).padStart(5, '0')}`;
    const batchRef = batchesCol.doc(batchId);
    const batchDigest = crypto.createHash('sha256')
      .update(`batch:v1:${genId}:${bIndex}:${chunkIds.join(',')}`)
      .digest('hex');

    const existingSnap = await batchRef.get();
    if (existingSnap.exists) {
      const existing = existingSnap.data();

      // Invariant: Completed batches must remain strictly immutable
      if (existing.status === 'completed') {
        if (existing.batchDigest !== batchDigest) {
          throw new Error(`COMPLETED_BATCH_CORRUPT_CONFLICT: Completed batch ${batchId} for generation ${genId} has digest ${existing.batchDigest}, expected ${batchDigest}. Mutation of completed batch forbidden.`);
        }
        // Exactly matches expected work plan; leave untouched
        continue;
      }

      const now = Date.now();
      const isLeaseActive = existing.status === 'processing' && existing.leaseExpiresAt && existing.leaseExpiresAt.toMillis() > now;

      // Invariant: Never modify submission IDs or batch digest while a batch is actively leased!
      if (isLeaseActive) {
        // Read and verify existing work-plan contents
        const hasMatchingDigest = existing.batchDigest === batchDigest;
        const hasMatchingIds = Array.isArray(existing.submissionIds) &&
          existing.submissionIds.length === chunkIds.length &&
          existing.submissionIds.every((id, idx) => id === chunkIds[idx]);

        if (!hasMatchingDigest || !hasMatchingIds) {
          throw new Error(`ACTIVE_BATCH_MUTATION_CONFLICT: Batch ${batchId} is actively leased to worker '${existing.leaseOwner}' (expires ${existing.leaseExpiresAt.toDate().toISOString()}) with conflicting content. Cannot mutate active batch.`);
        }

        // Active batch content matches expected plan perfectly; leave untouched
        console.log(`[Fanout Preparation] Batch ${batchId} is actively leased and matches expected work plan. Preserving active batch untouched.`);
        continue;
      }
    }

    // Default clean batch initialization (for new, pending, or expired batches)
    await batchRef.set({
      generationId: genId,
      batchIndex: bIndex,
      batchId: batchId,
      submissionIds: chunkIds,
      batchDigest: batchDigest,
      status: 'pending',
      expectedCount: chunkIds.length,
      completedCount: 0,
      skippedCount: 0,
      fanoutAttempts: 0,
      leaseOwner: null,
      leaseAcquiredAt: null,
      leaseExpiresAt: null,
      lastHeartbeatAt: null,
      createdAt: FirebaseFirestore.FieldValue.serverTimestamp(),
      updatedAt: FirebaseFirestore.FieldValue.serverTimestamp()
    }, { merge: true });
  }

  // 7. Bounded Cursor Paginated Deep Readback Verification
  await verifyPersistedBatches(genDocRef, genId, uniqueIds, totalBatches);

  // 8. Persist verified preparation state, work-plan digest, and release preparation lock
  await genDocRef.update({
    fanoutPreparationStatus: 'prepared',
    workPlanDigest: workPlanDigest,
    totalFanoutBatches: totalBatches,
    expectedSubmissionCount: uniqueIds.length,
    preparationOwner: null,
    preparationExpiresAt: null,
    updatedAt: FirebaseFirestore.FieldValue.serverTimestamp()
  });

  console.log(`[Fanout Preparation] Successfully prepared and verified ${totalBatches} batches (${uniqueIds.length} submissions) with workPlanDigest ${workPlanDigest} for generation ${genId}.`);
}

/**
 * Helper: Atomically acquires an exclusive 15-minute processing lease on a fanout batch.
 * Guarantees that two workers cannot simultaneously execute the same child batch.
 */
async function acquireBatchLease(db, batchRef, workerExecutionId) {
  return await db.runTransaction(async (tx) => {
    const bSnap = await tx.get(batchRef);
    if (!bSnap.exists) {
      return { acquired: false, reason: 'BATCH_NOT_FOUND' };
    }
    const bData = bSnap.data();

    if (bData.status === 'completed') {
      return { acquired: false, reason: 'ALREADY_COMPLETED' };
    }
    if (bData.status === 'manual_review') {
      return { acquired: false, reason: 'MANUAL_REVIEW' };
    }

    const now = Date.now();
    const leaseExpiresAtMillis = bData.leaseExpiresAt ? bData.leaseExpiresAt.toMillis() : 0;
    const isLeaseActive = bData.status === 'processing' && leaseExpiresAtMillis > now;

    if (isLeaseActive && bData.leaseOwner !== workerExecutionId) {
      // Actively held by another worker; do not acquire
      return { acquired: false, reason: 'ACTIVE_LEASE_HELD_BY_ANOTHER', owner: bData.leaseOwner };
    }

    // Acquire or renew 15-minute lease atomically
    const leaseExpiry = new Date(now + 15 * 60 * 1000);
    const batchAttempts = (bData.fanoutAttempts || 0) + 1;

    tx.update(batchRef, {
      status: 'processing',
      leaseOwner: workerExecutionId,
      leaseAcquiredAt: new Date(now),
      leaseExpiresAt: leaseExpiry,
      lastHeartbeatAt: new Date(now),
      fanoutAttempts: batchAttempts,
      lastAttemptAt: new Date(now),
      updatedAt: new Date(now)
    });

    return { acquired: true, batchData: bData };
  });
}

/**
 * Option A: Transactional Heartbeat / Lease Renewal Protocol.
 * Transactionally renews an active 15-minute batch lease when remaining time falls below 5 minutes.
 * Asserts atomic lease ownership, checks lease expiry, and enforces a maximum 60-minute window ceiling.
 */
export async function renewBatchLease(db, batchRef, workerExecutionId) {
  return await db.runTransaction(async (tx) => {
    const bSnap = await tx.get(batchRef);
    if (!bSnap.exists) {
      throw new Error(`BATCH_NOT_FOUND: Batch ${batchRef.id} missing during lease renewal`);
    }
    const bData = bSnap.data();
    const now = Date.now();

    // 1. Atomic Ownership Check: Superseded worker cannot renew an old lease
    if (bData.leaseOwner !== workerExecutionId) {
      throw new Error(`LEASE_RENEWAL_LOST_ABORT: Batch ${batchRef.id} lease owner mismatch (held by '${bData.leaseOwner}', expected '${workerExecutionId}'). Aborting.`);
    }

    // 2. Lease-Expiry Check: Lease must not have already expired
    if (!bData.leaseExpiresAt || bData.leaseExpiresAt.toMillis() <= now) {
      throw new Error(`LEASE_RENEWAL_EXPIRED_ABORT: Batch ${batchRef.id} lease expired before renewal could commit`);
    }

    // 3. Maximum Permitted Extension Check: Cap total lease duration at 60 minutes from leaseAcquiredAt
    const acquiredAt = bData.leaseAcquiredAt ? bData.leaseAcquiredAt.toMillis() : now;
    if (now - acquiredAt > 60 * 60 * 1000) {
      throw new Error(`MAX_LEASE_WINDOW_EXCEEDED: Batch ${batchRef.id} active lease exceeded maximum allowable 60-minute window`);
    }

    // 4. Extend lease by 15 minutes from current time
    const newExpiry = new Date(now + 15 * 60 * 1000);
    tx.update(batchRef, {
      leaseExpiresAt: newExpiry,
      lastHeartbeatAt: new Date(now),
      updatedAt: new Date(now)
    });

    return { renewed: true, newExpiry };
  });
}

/**
 * Executes or resumes generation-aware, conditional submission status fan-out
 * using bounded subcollection batches (/system_deployments/{generationId}/fanout_batches/{batchId}).
 *
 * Invariants Enforced (Phase 0.1.13):
 * 1. Transactional Distributed Lease Protocol with Heartbeat Renewal (Option A):
 *    Batches are claimed via acquireBatchLease(). Before each mini-chunk, if remaining lease < 5 minutes,
 *    renewBatchLease() atomically extends leaseExpiresAt by 15 minutes up to a 60-minute max window.
 *    If lease is lost, superseded, or expired, worker immediately aborts (LEASE_RENEWAL_LOST_ABORT).
 * 2. Global Deployment Fence: Mini-chunk transaction reads /system_config/deployment_state.
 *    - If genSeq < latestPublishedSequence: abort as STALE_GLOBAL_GENERATION_ABORT.
 *    - If genSeq > latestPublishedSequence: abort as UNPUBLISHED_GENERATION_ABORT.
 *    - If genSeq === latestPublishedSequence && genId !== latestPublishedGenerationId: fail closed as HALT_CONFLICT.
 * 3. Fail-Closed Product & Revision Validation: Missing, delisted, flagged, or inactive products fail closed.
 *    Submissions require positive integer approvedCatalogRevision <= capturedRevision.
 * 4. Submission Monotonic Sequence Guard & Conflict Resolution: Monotonic publishedGenerationSequence advance.
 * 5. Pure Transaction Callbacks: No external counter mutation inside db.runTransaction.
 * 6. Bounded Cursor Pagination: Child batches read via .orderBy('batchIndex', 'asc').limit(10).startAfter(lastBatchDoc).
 * 7. Atomic Batch Completion Transaction: Verifies lease ownership and expiry, marks child batch 'completed',
 *    clears lease, and increments parent generation summary counters in a SINGLE atomic transaction.
 * 8. Atomic Manual-Review Recovery with Structured Audit Logging:
 *    On 5 failed attempts or HALT_CONFLICT, atomically transitions child batch (status: 'manual_review'),
 *    parent generation (submissionFanoutStatus: 'manual_review'), and writes /admin_logs with a deterministic doc ID
 *    ('fanout_mr_' + genId + '_' + (batchId || 'root') + '_attempt_' + attempts) in ONE transaction.
 */
export async function executeSubmissionFanout(db, genDocRef, genData) {
  const genId = genData.generationId;
  const genSeq = genData.generationSequence;
  const capturedRev = genData.capturedRevision;
  let attempts = (genData.fanoutAttempts || 0) + 1;

  // Unique Worker Execution Identifier
  const runId = process.env.GITHUB_RUN_ID || 'local_worker';
  const runAttempt = process.env.GITHUB_RUN_ATTEMPT || '1';
  const workerExecutionId = `${runId}_${runAttempt}_${crypto.randomUUID()}`;

  await genDocRef.update({
    submissionFanoutStatus: 'in_progress',
    fanoutAttempts: attempts,
    lastFanoutAttemptAt: new Date(),
    updatedAt: new Date()
  });

  const BATCH_PAGE_SIZE = 10;
  let lastBatchDoc = null;
  let hasMoreBatches = true;
  let activeBatchRef = null;
  let activeBatchId = null;

  try {
    while (hasMoreBatches) {
      let batchQuery = genDocRef.collection('fanout_batches')
        .where('status', 'in', ['pending', 'processing'])
        .orderBy('batchIndex', 'asc')
        .limit(BATCH_PAGE_SIZE);

      if (lastBatchDoc) {
        batchQuery = batchQuery.startAfter(lastBatchDoc);
      }

      const batchesPageSnap = await batchQuery.get();

      if (batchesPageSnap.empty) {
        hasMoreBatches = false;
        break;
      }

      for (const batchDoc of batchesPageSnap.docs) {
        activeBatchRef = batchDoc.ref;
        activeBatchId = batchDoc.id;

        // 1. Transactional Lease Acquisition
        const leaseResult = await acquireBatchLease(db, batchDoc.ref, workerExecutionId);
        if (!leaseResult.acquired) {
          console.log(`[Fanout] Skipping batch ${activeBatchId}: Lease not acquired (${leaseResult.reason}${leaseResult.owner ? ` by ${leaseResult.owner}` : ''}).`);
          continue;
        }

        const batchData = leaseResult.batchData;
        const batchId = batchDoc.id;
        const submissionIds = batchData.submissionIds || [];

        let batchCompleted = 0;
        let batchSkipped = 0;
        const MINI_CHUNK_SIZE = 50;

        // 2. Process submission IDs in transactional mini-chunks with pure callbacks & lease renewal
        for (let i = 0; i < submissionIds.length; i += MINI_CHUNK_SIZE) {
          const chunkIds = submissionIds.slice(i, i + MINI_CHUNK_SIZE);

          // Option A: Heartbeat Lease Renewal before mini-chunk execution
          // If remaining lease time is under 5 minutes, renew lease transactionally
          const currentBatchSnap = await batchDoc.ref.get();
          if (currentBatchSnap.exists) {
            const currentBatch = currentBatchSnap.data();
            const remainingMs = currentBatch.leaseExpiresAt ? (currentBatch.leaseExpiresAt.toMillis() - Date.now()) : 0;
            if (remainingMs < 5 * 60 * 1000) {
              console.log(`[Fanout] Lease for batch ${batchId} has ${Math.round(remainingMs / 1000)}s remaining (< 5m). Renewing lease...`);
              await renewBatchLease(db, batchDoc.ref, workerExecutionId);
            }
          }

          const chunkResult = await db.runTransaction(async (transaction) => {
            // Pure callback: local outcome counters ONLY
            let chunkCompleted = 0;
            let chunkSkipped = 0;

            // Lease Ownership & Expiry Verification inside transaction
            const batchVerifySnap = await transaction.get(batchDoc.ref);
            if (!batchVerifySnap.exists) {
              throw new Error(`BATCH_NOT_FOUND: Batch ${batchId} was deleted during processing`);
            }
            const freshBatch = batchVerifySnap.data();
            if (freshBatch.leaseOwner !== workerExecutionId) {
              throw new Error(`LEASE_LOST_ABORT: Batch ${batchId} lease owner mismatch (held by ${freshBatch.leaseOwner}, expected ${workerExecutionId})`);
            }
            if (!freshBatch.leaseExpiresAt || freshBatch.leaseExpiresAt.toMillis() <= Date.now()) {
              throw new Error(`LEASE_EXPIRED_ABORT: Batch ${batchId} lease expired during mini-chunk execution`);
            }

            // Global Deployment State Verification at same consistency boundary
            const deployStateRef = db.collection('system_config').doc('deployment_state');
            const deployStateDoc = await transaction.get(deployStateRef);
            const deployState = deployStateDoc.exists ? deployStateDoc.data() : { latestPublishedSequence: 0, latestPublishedGenerationId: null };

            // Global Fence Evaluation
            if (genSeq < deployState.latestPublishedSequence) {
              throw new Error(`STALE_GLOBAL_GENERATION_ABORT: Generation ${genId} (seq ${genSeq}) < published sequence ${deployState.latestPublishedSequence}. Aborting fan-out.`);
            }
            if (genSeq > deployState.latestPublishedSequence) {
              throw new Error(`UNPUBLISHED_GENERATION_ABORT: Generation ${genId} (seq ${genSeq}) > published sequence ${deployState.latestPublishedSequence}. Awaiting publication.`);
            }
            if (genSeq === deployState.latestPublishedSequence && genId !== deployState.latestPublishedGenerationId) {
              throw new Error(`HALT_CONFLICT: Sequence collision at ${genSeq}. Generation ${genId} != published generation ${deployState.latestPublishedGenerationId}.`);
            }

            // Read submissions in mini-chunk
            const subRefs = chunkIds.map(id => db.collection('submissions').doc(id));
            const subSnapshots = await Promise.all(subRefs.map(ref => transaction.get(ref)));

            // Collect product IDs for product validation
            const productIdsToCheck = [];
            for (let j = 0; j < subSnapshots.length; j++) {
              const snap = subSnapshots[j];
              if (snap.exists) {
                const data = snap.data();
                const prodId = data.targetProductId || `prod_${data.repoNumericId}_${data.productData?.slug || ''}`;
                productIdsToCheck.push(prodId);
              } else {
                productIdsToCheck.push(null);
              }
            }

            const productRefs = productIdsToCheck.map(pId => pId ? db.collection('products').doc(pId) : null);
            const productSnapshots = await Promise.all(productRefs.map(pRef => pRef ? transaction.get(pRef) : Promise.resolve(null)));

            // Evaluate each submission deterministically (Fail-Closed)
            for (let j = 0; j < subSnapshots.length; j++) {
              const snap = subSnapshots[j];
              const subId = chunkIds[j];
              const ref = subRefs[j];
              const prodSnap = productSnapshots[j];

              if (!snap.exists) {
                chunkSkipped++;
                continue;
              }

              const currentSub = snap.data();

              // Submission Status Guard
              if (currentSub.status !== 'approved' && currentSub.status !== 'published') {
                chunkSkipped++;
                continue;
              }

              // Post-Capture Moderation Revision Fence (Fail-Closed)
              const approvalRev = currentSub.approvedCatalogRevision;
              if (typeof approvalRev !== 'number' || !Number.isInteger(approvalRev) || approvalRev <= 0) {
                console.warn(`[Fanout] Skipped submission ${subId}: missing, malformed, or non-positive approvedCatalogRevision (${approvalRev}).`);
                chunkSkipped++;
                continue;
              }
              if (approvalRev > capturedRev) {
                console.warn(`[Fanout] Skipped submission ${subId}: approvedCatalogRevision (${approvalRev}) > capturedRevision (${capturedRev}). Re-approved after capture.`);
                chunkSkipped++;
                continue;
              }

              // Product-Level Status Check (Fail-Closed)
              const targetProductId = currentSub.targetProductId || `prod_${currentSub.repoNumericId}_${currentSub.productData?.slug || ''}`;
              if (!prodSnap || !prodSnap.exists) {
                console.warn(`[Fanout] Skipped submission ${subId}: target product document ${targetProductId} does not exist.`);
                chunkSkipped++;
                continue;
              }
              const prodData = prodSnap.data();
              if (prodData.id !== targetProductId) {
                console.warn(`[Fanout] Skipped submission ${subId}: target product ID mismatch (${prodData.id} != ${targetProductId}).`);
                chunkSkipped++;
                continue;
              }
              if (prodData.status === 'delisted') {
                console.warn(`[Fanout] Skipped submission ${subId}: target product ${targetProductId} is delisted.`);
                chunkSkipped++;
                continue;
              }
              if (prodData.status === 'flagged') {
                console.warn(`[Fanout] Skipped submission ${subId}: target product ${targetProductId} is flagged.`);
                chunkSkipped++;
                continue;
              }
              if (prodData.status !== 'active') {
                console.warn(`[Fanout] Skipped submission ${subId}: target product status is not active (${prodData.status}).`);
                chunkSkipped++;
                continue;
              }

              // Monotonic Generation Sequence & Conflict Check
              const currentPublishedSeq = currentSub.publishedGenerationSequence || 0;

              if (currentPublishedSeq > genSeq) {
                chunkSkipped++;
                continue;
              }

              if (currentPublishedSeq === genSeq) {
                if (currentSub.publishedGenerationId !== genId) {
                  throw new Error(`HALT_CONFLICT: Submission ${subId} published sequence collision at ${genSeq}. Conflicting generation ${currentSub.publishedGenerationId}.`);
                }
                if (currentSub.status === 'published') {
                  chunkCompleted++;
                  continue;
                }
              }

              // Valid Monotonic Advance to published
              transaction.update(ref, {
                status: 'published',
                publishedGenerationId: genId,
                publishedGenerationSequence: genSeq,
                publishedProductId: targetProductId,
                'processingLease.leaseHolder': null,
                updatedAt: new Date()
              });
              chunkCompleted++;
            }

            return { chunkCompleted, chunkSkipped };
          });

          // Accumulate counts ONLY AFTER transaction commit succeeds
          batchCompleted += chunkResult.chunkCompleted;
          batchSkipped += chunkResult.chunkSkipped;
        }

        // 3. Atomic Batch Completion Transaction & Parent Counter Synchronization
        await db.runTransaction(async (tx) => {
          const freshBatchSnap = await tx.get(batchDoc.ref);
          if (!freshBatchSnap.exists) throw new Error(`Batch ${batchId} missing`);
          const freshBatchData = freshBatchSnap.data();

          if (freshBatchData.status === 'completed') {
            return; // Idempotent no-op
          }

          const now = Date.now();
          if (freshBatchData.leaseOwner !== workerExecutionId) {
            throw new Error(`LEASE_LOST_ABORT: Batch ${batchId} lease owner mismatch at completion (held by ${freshBatchData.leaseOwner}, expected ${workerExecutionId}).`);
          }
          if (!freshBatchData.leaseExpiresAt || freshBatchData.leaseExpiresAt.toMillis() < now) {
            throw new Error(`LEASE_EXPIRED_ABORT: Batch ${batchId} lease expired prior to completion commit.`);
          }

          const freshGenSnap = await tx.get(genDocRef);
          if (!freshGenSnap.exists) throw new Error(`Parent generation ${genId} missing`);

          tx.update(batchDoc.ref, {
            status: 'completed',
            completedCount: batchCompleted,
            skippedCount: batchSkipped,
            leaseOwner: null,
            leaseExpiresAt: null,
            completedAt: new Date(),
            updatedAt: new Date()
          });

          tx.update(genDocRef, {
            completedSubmissionCount: FirebaseFirestore.FieldValue.increment(batchCompleted + batchSkipped),
            completedFanoutBatches: FirebaseFirestore.FieldValue.increment(1),
            updatedAt: new Date()
          });
        });

        console.log(`[Fanout] Atomically completed batch ${batchId} (${batchCompleted} completed, ${batchSkipped} skipped) by worker ${workerExecutionId}.`);
      }

      if (batchesPageSnap.size < BATCH_PAGE_SIZE) {
        hasMoreBatches = false;
      } else {
        lastBatchDoc = batchesPageSnap.docs[batchesPageSnap.docs.length - 1];
      }
    }

    // 4. Finalize Generation if All Batches are Confirmed Completed
    const remainingBatchesSnap = await genDocRef.collection('fanout_batches')
      .where('status', 'in', ['pending', 'processing', 'manual_review'])
      .limit(1)
      .get();

    if (remainingBatchesSnap.empty) {
      await genDocRef.update({
        submissionFanoutStatus: 'completed',
        reconciliationStatus: 'reconciled',
        status: 'finalized',
        finalizedAt: new Date(),
        updatedAt: new Date()
      });
      console.log(`[Fanout] All fan-out batches completed for generation ${genId}. Finalized.`);
    }

  } catch (err) {
    console.error(`[Fanout] Error processing fan-out for generation ${genId}: ${err.message}`);

    if (attempts >= 5 || (err.message && err.message.includes('HALT_CONFLICT'))) {
      console.error(`[Fanout] Fatal error or max retries reached. Atomically transitioning generation ${genId}, active batch, and writing deterministic /admin_logs in ONE transaction.`);

      // Deterministic Audit Event ID: idempotent across transaction retries
      const auditEventId = `fanout_mr_${genId}_${activeBatchId || 'root'}_attempt_${attempts}`;
      const auditLogRef = db.collection('admin_logs').doc(auditEventId);

      // Single Atomic Transaction: child batch, parent generation, and /admin_logs record
      // Strict Firestore Read-Before-Write across all target documents
      await db.runTransaction(async (tx) => {
        // READ 1: Parent generation document
        const genSnap = await tx.get(genDocRef);

        // READ 2: Active batch document (if present)
        let bSnap = null;
        if (activeBatchRef) {
          bSnap = await tx.get(activeBatchRef);
        }

        // READ 3: Deterministic audit log document
        const auditSnap = await tx.get(auditLogRef);

        const now = new Date();

        // WRITE 1: Update active child batch if not already completed
        if (bSnap && bSnap.exists && bSnap.data().status !== 'completed') {
          tx.update(activeBatchRef, {
            status: 'manual_review',
            leaseOwner: null,
            leaseExpiresAt: null,
            lastError: err.message,
            failedAt: now,
            updatedAt: now
          });
        }

        // WRITE 2: Update parent generation if it exists
        if (genSnap.exists) {
          tx.update(genDocRef, {
            submissionFanoutStatus: 'manual_review',
            reconciliationStatus: 'manual_review',
            lastFanoutError: `Fan-out failed: ${err.message}`,
            manualReviewAt: now,
            updatedAt: now
          });
        }

        // WRITE 3: Write immutable deterministic audit log entry (idempotent on retries)
        if (!auditSnap.exists) {
          tx.set(auditLogRef, {
            eventId: auditEventId,
            action: 'SUBMISSION_FANOUT_MANUAL_REVIEW',
            generationId: genId,
            batchId: activeBatchId || null,
            workerExecutionId: workerExecutionId,
            attempts: attempts,
            error: err.message,
            createdAt: now,
            updatedAt: now
          });
        }
      });

      console.log(`[Fanout] Atomically transitioned generation ${genId} and batch ${activeBatchId} to manual_review with deterministic audit log ${auditEventId}.`);
    } else {
      await genDocRef.update({
        submissionFanoutStatus: 'partial',
        lastFanoutError: err.message,
        updatedAt: new Date()
      });
    }
  }
}```

---

## 4. Hardened Security Rules (`firestore.rules`)

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isAuthenticated() {
      return request.auth != null;
    }

    function isOwner(userId) {
      return isAuthenticated() && request.auth.uid == userId;
    }

    function isFreshlyAuthenticated() {
      return isAuthenticated() 
        && request.auth.token.firebase.sign_in_provider == 'github.com'
        && ('github.com' in request.auth.token.firebase.identities)
        && request.auth.token.auth_time != null
        && ((request.time.toMillis() / 1000) - request.auth.token.auth_time < 300)
        && (request.auth.token.auth_time <= (request.time.toMillis() / 1000) + 5);
    }

    // Action-specific payload validators
    function validModerationPayload(action, payload) {
      return payload is map && (
        (action == 'APPROVE_SUBMISSION' && validApprovePayload(payload)) ||
        (action == 'REJECT_SUBMISSION' && validRejectPayload(payload)) ||
        (action == 'VERIFY_DEVELOPER' && validVerifyDevPayload(payload)) ||
        (action == 'REVOKE_DEVELOPER' && validRevokeDevPayload(payload)) ||
        (action == 'DELIST_PRODUCT' && validDelistPayload(payload)) ||
        (action == 'FEATURE_PRODUCT' && validFeaturePayload(payload))
      );
    }

    function validApprovePayload(payload) {
      return payload.keys().hasOnly(['reviewNotes'])
        && (!('reviewNotes' in payload) || (payload.reviewNotes is string && payload.reviewNotes.size() <= 1000));
    }

    function validRejectPayload(payload) {
      return payload.keys().hasOnly(['rejectionReason', 'reviewNotes'])
        && ('rejectionReason' in payload)
        && payload.rejectionReason is string
        && payload.rejectionReason.size() >= 10
        && payload.rejectionReason.size() <= 1000
        && (!('reviewNotes' in payload) || (payload.reviewNotes is string && payload.reviewNotes.size() <= 1000));
    }

    function validVerifyDevPayload(payload) {
      return payload.keys().hasOnly(['verificationReason', 'evidenceUrl'])
        && ('verificationReason' in payload)
        && payload.verificationReason is string
        && payload.verificationReason.size() >= 10
        && payload.verificationReason.size() <= 500
        && (!('evidenceUrl' in payload) || (payload.evidenceUrl is string && payload.evidenceUrl.size() <= 200));
    }

    function validRevokeDevPayload(payload) {
      return payload.keys().hasOnly(['revocationReason'])
        && ('revocationReason' in payload)
        && payload.revocationReason is string
        && payload.revocationReason.size() >= 10
        && payload.revocationReason.size() <= 500;
    }

    function validDelistPayload(payload) {
      return payload.keys().hasOnly(['delistReason', 'noticePeriodDays'])
        && ('delistReason' in payload)
        && payload.delistReason is string
        && payload.delistReason.size() >= 10
        && payload.delistReason.size() <= 1000
        && (!('noticePeriodDays' in payload) || (payload.noticePeriodDays is number && payload.noticePeriodDays >= 0 && payload.noticePeriodDays <= 30));
    }

    function validFeaturePayload(payload) {
      return payload.keys().hasOnly(['featuredRank', 'featureNotes'])
        && ('featuredRank' in payload)
        && payload.featuredRank is number
        && payload.featuredRank >= 1
        && payload.featuredRank <= 100
        && (!('featureNotes' in payload) || (payload.featureNotes is string && payload.featureNotes.size() <= 500));
    }

    // Exhaustive Nested productData Schema Validators (Section 5 Invariants)
    function isValidProductData(p) {
      return p is map
        && p.keys().hasAll([
          'name', 'slug', 'repoUrl', 'repoOwner', 'repoName', 'releaseTag',
          'selectedAsset', 'shortDescription', 'fullDescription', 'category',
          'iconUrl', 'screenshots', 'supportedPlatforms', 'version', 'license'
        ])
        && p.keys().hasOnly([
          'name', 'slug', 'repoUrl', 'repoOwner', 'repoName', 'releaseTag',
          'selectedAsset', 'shortDescription', 'fullDescription', 'category',
          'iconUrl', 'screenshots', 'supportedPlatforms', 'version', 'license',
          'liveDemoUrl'
        ])
        // Basic Information
        && p.name is string && p.name.size() >= 3 && p.name.size() <= 60
        && p.slug is string && p.slug.size() >= 3 && p.slug.size() <= 30
        && p.slug.matches('^[a-z0-9]+(-[a-z0-9]+)*$')
        && p.repoUrl is string && p.repoUrl.size() <= 200
        && p.repoUrl.matches('^https://github\\.com/[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+$')
        && p.repoOwner is string && p.repoOwner.size() >= 1 && p.repoOwner.size() <= 39
        && p.repoOwner.matches('^[A-Za-z0-9]([A-Za-z0-9-]*[A-Za-z0-9])?$')
        && p.repoName is string && p.repoName.size() >= 1 && p.repoName.size() <= 100
        && p.repoName.matches('^[A-Za-z0-9_.-]+$')
        // Strict consistency between repoUrl and repoOwner/repoName
        && p.repoUrl == 'https://github.com/' + p.repoOwner + '/' + p.repoName
        && p.releaseTag is string && p.releaseTag.size() >= 1 && p.releaseTag.size() <= 50
        && p.shortDescription is string && p.shortDescription.size() >= 10 && p.shortDescription.size() <= 200
        && p.fullDescription is string && p.fullDescription.size() >= 20 && p.fullDescription.size() <= 10000
        && p.category is string && p.category in [
          'developer-tools', 'utilities', 'productivity', 'graphics-media',
          'games', 'education', 'security', 'other'
        ]
        // Version: Structural format check in Rules (1-32 chars); authoritative SemVer 2.0.0 validated by worker
        && p.version is string && p.version.size() >= 1 && p.version.size() <= 32
        && p.version.matches('^[0-9A-Za-z.+-]+$')
        // License: Structural format check in Rules (1-50 chars); authoritative SPDX catalog validated by worker
        && p.license is string && p.license.size() >= 1 && p.license.size() <= 50
        // Selected Asset Map (all 5 fields validated)
        && isValidSelectedAsset(p.selectedAsset)
        // Media (iconUrl + screenshots list elements)
        && p.iconUrl is string && (p.iconUrl == '' || (p.iconUrl.size() <= 500 && p.iconUrl.matches('^https://.*')))
        && isValidScreenshots(p.screenshots)
        // Platforms (enum list validation with explicit non-duplication check)
        && isValidPlatforms(p.supportedPlatforms)
        // Optional liveDemoUrl
        && (!('liveDemoUrl' in p) || p.liveDemoUrl == null || (p.liveDemoUrl is string && p.liveDemoUrl.size() <= 500 && p.liveDemoUrl.matches('^https://.*')));
    }

    function isValidSelectedAsset(asset) {
      return asset is map
        && asset.keys().hasAll(['name', 'extension', 'sizeBytes', 'downloadUrl', 'assetId'])
        && asset.keys().hasOnly(['name', 'extension', 'sizeBytes', 'downloadUrl', 'assetId'])
        && asset.name is string && asset.name.size() >= 1 && asset.name.size() <= 128
        && asset.extension is string && asset.extension.size() >= 1 && asset.extension.size() <= 16
        && asset.sizeBytes is int && asset.sizeBytes > 0 && asset.sizeBytes <= 524288000 // 500 MiB limit
        && asset.downloadUrl is string && asset.downloadUrl.size() <= 500 && asset.downloadUrl.matches('^https://github\\.com/[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+/releases/download/.*')
        && asset.assetId is int && asset.assetId > 0;
    }

    function isValidScreenshots(shots) {
      return shots is list
        && shots.size() <= 6
        && (shots.size() < 1 || (shots[0] is string && shots[0].size() <= 500 && shots[0].matches('^https://.*')))
        && (shots.size() < 2 || (shots[1] is string && shots[1].size() <= 500 && shots[1].matches('^https://.*')))
        && (shots.size() < 3 || (shots[2] is string && shots[2].size() <= 500 && shots[2].matches('^https://.*')))
        && (shots.size() < 4 || (shots[3] is string && shots[3].size() <= 500 && shots[3].matches('^https://.*')))
        && (shots.size() < 5 || (shots[4] is string && shots[4].size() <= 500 && shots[4].matches('^https://.*')))
        && (shots.size() < 6 || (shots[5] is string && shots[5].size() <= 500 && shots[5].matches('^https://.*')));
    }

    function isValidPlatforms(platforms) {
      return platforms is list
        && platforms.size() >= 1
        && platforms.size() <= 6
        && platforms.hasOnly(['windows', 'macos', 'linux', 'android', 'ios', 'web'])
        && arePlatformsUnique(platforms);
    }

    function arePlatformsUnique(p) {
      return p.size() == 1
        || (p.size() == 2 && p[0] != p[1])
        || (p.size() == 3 && p[0] != p[1] && p[0] != p[2] && p[1] != p[2])
        || (p.size() == 4 && p[0] != p[1] && p[0] != p[2] && p[0] != p[3] && p[1] != p[2] && p[1] != p[3] && p[2] != p[3])
        || (p.size() == 5 && p[0] != p[1] && p[0] != p[2] && p[0] != p[3] && p[0] != p[4] && p[1] != p[2] && p[1] != p[3] && p[1] != p[4] && p[2] != p[3] && p[2] != p[4] && p[3] != p[4])
        || (p.size() == 6 && p[0] != p[1] && p[0] != p[2] && p[0] != p[3] && p[0] != p[4] && p[0] != p[5] && p[1] != p[2] && p[1] != p[3] && p[1] != p[4] && p[1] != p[5] && p[2] != p[3] && p[2] != p[4] && p[2] != p[5] && p[3] != p[4] && p[3] != p[5] && p[4] != p[5]);
    }

    // -------------------------------------------------------------------------
    // 1. Private User Documents (/users/{userId})
    // -------------------------------------------------------------------------
    match /users/{userId} {
      allow read: if isOwner(userId);
      allow create: if isOwner(userId)
        && request.resource.data.keys().hasOnly(['uid', 'email', 'displayName', 'avatarUrl', 'bio', 'settings', 'createdAt', 'updatedAt'])
        && request.resource.data.uid == userId
        && request.resource.data.createdAt == request.time
        && request.resource.data.updatedAt == request.time;
      allow update: if isOwner(userId)
        && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['displayName', 'avatarUrl', 'bio', 'settings', 'updatedAt'])
        && request.resource.data.updatedAt == request.time;
      allow delete: if false;
    }

    // -------------------------------------------------------------------------
    // 2. Public Developer Profiles (/public_profiles/{userId})
    // -------------------------------------------------------------------------
    match /public_profiles/{userId} {
      allow read: if true;
      allow create: if isOwner(userId)
        && request.resource.data.keys().hasOnly(['userId', 'displayName', 'githubUsername', 'avatarUrl', 'bio', 'isVerifiedDeveloper', 'publishedProductIds', 'createdAt', 'updatedAt'])
        && request.resource.data.userId == userId
        && request.resource.data.isVerifiedDeveloper == false
        && request.resource.data.publishedProductIds == []
        && request.resource.data.createdAt == request.time
        && request.resource.data.updatedAt == request.time;
      allow update: if isOwner(userId)
        && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['displayName', 'avatarUrl', 'bio', 'updatedAt'])
        && request.resource.data.updatedAt == request.time;
      allow delete: if false;
    }

    // -------------------------------------------------------------------------
    // 3. Submissions (/submissions/{submissionId})
    // -------------------------------------------------------------------------
    match /submissions/{submissionId} {
      allow read: if isAuthenticated() && resource.data.developerUid == request.auth.uid;
      allow create: if isAuthenticated() 
        && request.resource.data.keys().hasOnly([
          'id', 'developerUid', 'submissionSource', 'repoNumericId', 'repoId',
          'targetProductId', 'productData', 'status', 'claimStatus',
          'createdAt', 'updatedAt'
        ])
        && request.resource.data.id == submissionId
        && request.resource.data.developerUid == request.auth.uid
        && request.resource.data.status == 'queued'
        && request.resource.data.claimStatus == 'claimed'
        && request.resource.data.submissionSource == 'dashboard'
        && isValidProductData(request.resource.data.productData)
        && request.resource.data.createdAt == request.time
        && request.resource.data.updatedAt == request.time;
      allow update: if isOwner(resource.data.developerUid)
        && (resource.data.status == 'queued' || resource.data.status == 'rejected')
        && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['productData', 'status', 'updatedAt'])
        && request.resource.data.status == 'queued'
        && isValidProductData(request.resource.data.productData)
        && request.resource.data.updatedAt == request.time;
      allow delete: if false;
    }

    // -------------------------------------------------------------------------
    // 4. Claim Requests (/claim_requests/{requestId})
    // -------------------------------------------------------------------------
    match /claim_requests/{requestId} {
      allow read: if isAuthenticated() && resource.data.requesterUid == request.auth.uid;
      allow create: if isAuthenticated()
        && request.resource.data.keys().hasOnly(['requestId', 'requesterUid', 'targetSubmissionId', 'status', 'createdAt'])
        && request.resource.data.requestId == requestId
        && request.resource.data.requesterUid == request.auth.uid
        && request.resource.data.targetSubmissionId is string
        && request.resource.data.targetSubmissionId.size() > 0
        && request.resource.data.targetSubmissionId.size() <= 128
        && request.resource.data.status == 'pending'
        && request.resource.data.createdAt == request.time;
      allow update, delete: if false;
    }

    // -------------------------------------------------------------------------
    // 5. Moderation Requests (/moderation_requests/{requestId})
    // -------------------------------------------------------------------------
    match /moderation_requests/{requestId} {
      allow read: if isAuthenticated() && resource.data.requestedByUid == request.auth.uid;
      allow create: if isFreshlyAuthenticated()
        && request.resource.data.keys().hasOnly([
          'id', 'action', 'targetId', 'targetDocumentPath', 'payload',
          'requestedByUid', 'githubUserId', 'stepUpProof', 'status', 'createdAt'
        ])
        && request.resource.data.id == requestId
        && request.resource.data.requestedByUid == request.auth.uid
        && request.resource.data.githubUserId == request.auth.token.firebase.identities['github.com'][0]
        && request.resource.data.action in [
          'APPROVE_SUBMISSION', 'REJECT_SUBMISSION', 'VERIFY_DEVELOPER',
          'REVOKE_DEVELOPER', 'DELIST_PRODUCT', 'FEATURE_PRODUCT'
        ]
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
        && request.resource.data.stepUpProof is map
        && request.resource.data.stepUpProof.keys().hasOnly([
          'providerId', 'authTime', 'actionNonce', 'targetDocumentPath', 'actionHash'
        ])
        && request.resource.data.stepUpProof.providerId == 'github.com'
        && request.resource.data.stepUpProof.authTime == request.auth.token.auth_time
        && request.resource.data.stepUpProof.actionNonce is string
        && request.resource.data.stepUpProof.actionNonce.size() >= 16
        && request.resource.data.stepUpProof.actionNonce.size() <= 128
        && request.resource.data.stepUpProof.targetDocumentPath == request.resource.data.targetDocumentPath
        && request.resource.data.stepUpProof.actionHash is string
        && request.resource.data.stepUpProof.actionHash.size() == 64
        && request.resource.data.status == 'pending'
        && request.resource.data.createdAt == request.time;
      allow update, delete: if false;
    }

    // -------------------------------------------------------------------------
    // 6. Consumed Step-Up Nonces (/consumed_stepup_nonces/{nonceHash})
    // -------------------------------------------------------------------------
    match /consumed_stepup_nonces/{nonceHash} {
      allow read, write: if false; // Worker only via Admin SDK
    }

    // -------------------------------------------------------------------------
    // 7. Products (/products/{productId})
    // -------------------------------------------------------------------------
    match /products/{productId} {
      allow read: if resource.data.status == 'active';
      allow write: if false; // Worker only via Admin SDK
    }

    // -------------------------------------------------------------------------
    // 8. Categories (/categories/{categoryId})
    // -------------------------------------------------------------------------
    match /categories/{categoryId} {
      allow read: if true;
      allow write: if false;
    }

    // -------------------------------------------------------------------------
    // 9. Abuse Reports (/reports/{reportId})
    // -------------------------------------------------------------------------
    match /reports/{reportId} {
      allow create: if isAuthenticated()
        && request.resource.data.keys().hasOnly([
          'reportId', 'reporterUid', 'targetType', 'targetId',
          'reason', 'details', 'status', 'createdAt'
        ])
        && request.resource.data.reportId == reportId
        && request.resource.data.reporterUid == request.auth.uid
        && request.resource.data.targetType in ['product', 'submission', 'user']
        && request.resource.data.targetId is string
        && request.resource.data.targetId.size() > 0
        && request.resource.data.targetId.size() <= 128
        && request.resource.data.reason in ['malware', 'copyright', 'misleading', 'broken', 'other']
        && request.resource.data.details is string
        && request.resource.data.details.size() <= 2000
        && request.resource.data.status == 'open'
        && request.resource.data.createdAt == request.time;
      allow read, update, delete: if false;
    }

    // -------------------------------------------------------------------------
    // 10. Internal Collections & State (Default-Deny for Clients)
    // -------------------------------------------------------------------------
    match /admin_logs/{logId} {
      allow read, write: if false;
    }
    match /system_deployments/{generationId} {
      allow read, write: if false;
      match /fanout_batches/{batchId} {
        allow read, write: if false;
      }
    }
    match /system_config/deployment_state {
      allow read, write: if false;
    }
    match /system_config/catalog_state {
      allow read, write: if false;
    }
    match /system_config/global_settings {
      allow read: if true;
      allow write: if false;
    }
  }
}
```

---

## 5. Required Composite Indexes (`firestore.indexes.json`)

To ensure efficient queries for batch workers, reconcilers, and administration without table scans, the following composite indexes must be defined:

```json
{
  "indexes": [
    {
      "collectionGroup": "system_deployments",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "submissionFanoutStatus", "order": "ASCENDING" },
        { "fieldPath": "generationSequence", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "system_deployments",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "reconciliationStatus", "order": "ASCENDING" },
        { "fieldPath": "generationSequence", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "system_deployments",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "submissionFanoutStatus", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "fanout_batches",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "batchIndex", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "fanout_batches",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "leaseExpiresAt", "order": "ASCENDING" },
        { "fieldPath": "batchIndex", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "fanout_batches",
      "queryScope": "COLLECTION_GROUP",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "submissions",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "publishedGenerationSequence", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "submissions",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "developerUid", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "moderation_requests",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "ASCENDING" }
      ]
    }
  ],
  "fieldOverrides": []
}
```
