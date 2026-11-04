# GitStore — Cloud Firestore Database Schema & Data Architecture

**Document Version:** 1.11.0  
**Phase:** Phase 0.1.11 — Fan-Out Generation Fencing, Idempotent Retry Accounting & Schema Truthfulness  
**Status:** IN_REVIEW — Awaiting Explicit User Approval  
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
7. **Global Generation Fencing, Pure Transaction Callbacks & 2-Tier Finalization:** When finalizing deployment state, the worker executes a 2-tier process governed by strict generation fences:
   - **Tier 1 Publication Transaction:** Reads current versions of `/system_config/catalog_state`, `/system_config/deployment_state`, and `/system_deployments/{generationId}` BEFORE any writes. Inside the transaction, it verifies that child batch preparation is fully complete (`fanoutPreparationStatus === 'prepared'`), evaluates generation ordering (`evaluateGenerationOrdering`), and enforces catalog revision monotonicity. Stale generations abort immediately (`stale_aborted`) with zero state modifications.
   - **Tier 2 Submission Fan-Out Execution:** Fan-out work is partitioned into bounded child documents (`/system_deployments/{generationId}/fanout_batches/{batchId}`) capped at 400 IDs each (~55 KB << 1 MiB). Mini-chunk transactions read `/system_config/deployment_state` inside the transaction boundary; if `genSeq < latestPublishedSequence`, the fan-out aborts as stale (`STALE_GLOBAL_GENERATION_ABORT`), preventing older generations from ever modifying unreached records. Per-submission checks enforce `approvedCatalogRevision <= capturedRevision`, verify that products are not `delisted` or `flagged` in `/products/{productId}`, fail closed on equal-sequence conflicting generation IDs (`HALT_CONFLICT`), and acknowledge idempotent replays. Transaction callbacks are strictly pure (zero outer counter mutations), and child batch completion atomically updates parent summary counters in a single transaction.
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
  fanoutPreparationStatus: 'pending' | 'prepared'; // Work-plan preparation marker (must be 'prepared' before Tier 1 publication)
  submissionFanoutStatus: 'pending' | 'in_progress' | 'completed' | 'partial' | 'failed' | 'manual_review';
  expectedSubmissionCount: number;   // Total expected submissions across all batches (O(1) scalar)
  completedSubmissionCount: number;  // Confirmed successfully processed/skipped submissions (O(1) scalar)
  totalFanoutBatches: number;        // Total deterministic batch documents in subcollection (O(1) scalar)
  completedFanoutBatches: number;    // Number of completed batches in subcollection (O(1) scalar)
  fanoutAttempts: number;            // Number of fan-out execution attempts across the generation
  lastFanoutAttemptAt?: FirebaseFirestore.Timestamp | null;
  lastFanoutError?: string | null;

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
  status: 'pending' | 'processing' | 'completed' | 'manual_review';
  expectedCount: number;             // submissionIds.length (<= 400)
  completedCount: number;            // Confirmed updated or idempotently verified submissions
  skippedCount: number;              // Submissions skipped due to newer generation sequence, revision fence, or product delisting
  fanoutAttempts: number;            // Number of execution attempts for this specific batch
  leaseOwner: string | null;         // Worker run ID holding active processing lease (or null)
  leaseExpiresAt: FirebaseFirestore.Timestamp | null; // Atomic 15-minute lease expiry timestamp
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

    // 4a. Error-Classified Commit Recovery for missing catalogCommitSha
    if (!gen.catalogCommitSha) {
      console.log(`[Reconciler] Generation ${gen.generationId} missing catalogCommitSha. Scanning origin/main history...`);
      let recoveredSha = null;
      let isSearchIncomplete = false;

      // Primary: Unshallow Git log
      try {
        const rawLog = execSync('git log --format="%H" origin/main -- data/deployment-manifest.json', { encoding: 'utf8' }).trim();
        const candidateShas = rawLog ? rawLog.split('\n') : [];
        for (const sha of candidateShas) {
          if (!/^[0-9a-f]{40}$/i.test(sha)) continue;
          try {
            execSync(`git merge-base --is-ancestor ${sha} origin/main`);
            const blobContent = execSync(`git show ${sha}:data/deployment-manifest.json`, { encoding: 'utf8' });
            const manifest = JSON.parse(blobContent);
            if (manifest.generationId === gen.generationId && 
                manifest.generationSequence === gen.generationSequence &&
                manifest.catalogDigest === gen.catalogDigest) {
              recoveredSha = sha;
              break;
            }
          } catch (e) { /* candidate did not match */ }
        }
      } catch (localGitErr) {
        console.warn(`[Reconciler] Local git log search failed: ${localGitErr.message}`);
      }

      // Fallback: GitHub Commits API with Full Pagination & Strict Error Classification
      if (!recoveredSha) {
        let currentUrl = `https://api.github.com/repos/Rohitpvt/GitStore/commits?sha=main&path=data/deployment-manifest.json&per_page=100`;
        let hasCorruptCandidates = false;
        let isFatalAuthError = false;

        while (currentUrl && !recoveredSha) {
          try {
            const apiCommits = await octokit.request(`GET ${currentUrl}`);
            for (const item of apiCommits.data) {
              if (!/^[0-9a-f]{40}$/i.test(item.sha)) continue;
              try {
                const fileContentRes = await octokit.request('GET /repos/{owner}/{repo}/contents/{path}', {
                  owner: 'Rohitpvt',
                  repo: 'GitStore',
                  path: 'data/deployment-manifest.json',
                  ref: item.sha
                });

                if (!fileContentRes.data || !fileContentRes.data.content) {
                  throw new Error('API_MALFORMED_RESPONSE: Missing content field in GitHub Contents API response');
                }

                const decoded = Buffer.from(fileContentRes.data.content, 'base64').toString('utf8');
                const manifest = JSON.parse(decoded);

                if (manifest.generationId === gen.generationId && 
                    manifest.generationSequence === gen.generationSequence &&
                    manifest.catalogDigest === gen.catalogDigest) {
                  recoveredSha = item.sha;
                  break;
                }
              } catch (blobErr) {
                // Strict Error Classification (Section 4 Invariants)
                const errorInfo = classifyCommitInspectionError(blobErr);

                if (errorInfo.classification === 'CANDIDATE_NO_MANIFEST') {
                  // Benign candidate skip: Commit did not touch manifest blob
                  continue;
                } else if (errorInfo.classification === 'CANDIDATE_CORRUPT_MANIFEST') {
                  // Corrupt manifest JSON: Record corruption; DO NOT treat as empty or missing!
                  console.warn(`[Reconciler] Corrupt manifest in commit ${item.sha}: ${blobErr.message}`);
                  hasCorruptCandidates = true;
                  continue;
                } else if (errorInfo.isFatal) {
                  // Fatal auth error: Must halt immediately and alert administrator
                  console.error(`[Reconciler] Fatal commit inspection failure (${errorInfo.classification}): ${blobErr.message}`);
                  isSearchIncomplete = true;
                  isFatalAuthError = true;
                  break;
                } else if (errorInfo.isIncomplete) {
                  // Upstream 5xx, 429/403 rate limit, network timeout, or connection reset
                  console.warn(`[Reconciler] Incomplete commit inspection (${errorInfo.classification}) for SHA ${item.sha}: ${blobErr.message}`);
                  isSearchIncomplete = true;
                  break;
                }
              }
            }

            if (isSearchIncomplete || recoveredSha || isFatalAuthError) break;

            const linkHeader = apiCommits.headers['link'];
            const nextMatch = linkHeader ? linkHeader.match(/<([^>]+)>;\s*rel="next"/) : null;
            currentUrl = nextMatch ? nextMatch[1] : null;
          } catch (pageErr) {
            console.warn(`[Reconciler] Commits API pagination failed: ${pageErr.message}`);
            isSearchIncomplete = true;
            break;
          }
        }

        if (recoveredSha) {
          console.log(`[Reconciler] Recovered catalogCommitSha ${recoveredSha} for generation ${gen.generationId}.`);
          gen.catalogCommitSha = recoveredSha;
          await genDoc.ref.update({
            catalogCommitSha: recoveredSha,
            reconciliationStatus: 'sha_recovered',
            updatedAt: new Date()
          });
        } else if (isFatalAuthError) {
          console.error(`[Reconciler] Fatal authentication failure on GitHub API. Retaining generation ${gen.generationId} in pending_retry.`);
          await genDoc.ref.update({ reconciliationStatus: 'pending_retry', updatedAt: new Date() });
          break; // Stop reconciler loop to avoid burning failed auth attempts
        } else if (isSearchIncomplete) {
          console.warn(`[Reconciler] Search incomplete due to network/rate-limit interruptions. Retaining generation ${gen.generationId} as pending_retry.`);
          await genDoc.ref.update({ reconciliationStatus: 'pending_retry', updatedAt: new Date() });
          continue;
        } else if (hasCorruptCandidates) {
          console.warn(`[Reconciler] Remote history contains corrupt manifest JSON. Routing generation ${gen.generationId} to manual_review.`);
          await genDoc.ref.update({
            status: 'failed',
            reconciliationStatus: 'manual_review',
            failureReason: 'corrupt_manifest_encountered_on_remote',
            retryEligibility: false,
            updatedAt: new Date()
          });
          continue;
        } else {
          // Exhaustive scan completed with ZERO network interruptions, ZERO auth errors, and ZERO corrupt candidates
          console.warn(`[Reconciler] Exhaustive search confirmed EMPTY_SEARCH for generation ${gen.generationId}.`);
          await genDoc.ref.update({
            status: 'failed',
            failureReason: 'commit_not_found_on_remote',
            reconciliationStatus: 'abandoned',
            retryEligibility: false,
            updatedAt: new Date()
          });
          continue;
        }
      }
    }

    // 4b. Dedicated Pages Status Check ('succeed')
    let isPagesSucceed = false;
    let resolvedReference = gen.pagesDeploymentReference || gen.catalogCommitSha;

    try {
      const pagesRes = await octokit.request('GET /repos/{owner}/{repo}/pages/deployments/{pages_deployment_id}', {
        owner: 'Rohitpvt',
        repo: 'GitStore',
        pages_deployment_id: resolvedReference
      });

      if (pagesRes.status === 200) {
        const deployData = pagesRes.data;
        if (deployData.status === 'succeed') {
          isPagesSucceed = true;
          await genDoc.ref.update({
            pagesDeploymentReference: resolvedReference,
            lastObservedPagesStatus: 'succeed'
          });
        } else if (deployData.status === 'queued' || deployData.status === 'in_progress') {
          await genDoc.ref.update({ reconciliationStatus: 'pages_dispatched', lastObservedPagesStatus: deployData.status });
          continue;
        } else if (deployData.status === 'errored') {
          await genDoc.ref.update({ reconciliationStatus: 'pending_retry', lastObservedPagesStatus: 'errored' });
          continue;
        } else {
          console.warn(`[Reconciler] Unrecognized Pages deployment status '${deployData.status}'. Treating as non-terminal.`);
          continue;
        }
      }
    } catch (pagesErr) {
      if (pagesErr.status === 404) {
        // Case B Recovery: Clean Staging from exact commit
        console.log(`[Reconciler] Case B: No Pages deployment found for commit ${gen.catalogCommitSha}. Executing recovery staging.`);
        try {
          if (!/^[0-9a-f]{40}$/i.test(gen.catalogCommitSha)) {
            throw new Error(`Invalid catalogCommitSha: ${gen.catalogCommitSha}`);
          }
          // Single authoritative staging procedure:
          execSync(`rm -rf site/ && mkdir -p site/data site/assets`);
          execSync(`git archive ${gen.catalogCommitSha} index.html admin.html dashboard.html submit.html product.html explore.html assets data | tar -x -C site/`);
          
          const stagedDigest = computeCatalogDigest('site/data');
          if (stagedDigest !== gen.catalogDigest) {
            throw new Error(`STAGED_DIGEST_MISMATCH: staged ${stagedDigest} !== expected ${gen.catalogDigest}`);
          }

          await genDoc.ref.update({
            reconciliationStatus: 'pages_dispatched',
            stagedDigest: stagedDigest,
            updatedAt: new Date()
          });

          handoffResult = {
            commitNeeded: false,
            deployNeeded: true,
            generationId: gen.generationId,
            generationSequence: gen.generationSequence,
            catalogCommitSha: gen.catalogCommitSha,
            catalogDigest: gen.catalogDigest,
            capturedRevision: gen.capturedRevision
          };
          emitStepOutput('commit_needed', 'false');
          emitStepOutput('deploy_needed', 'true');
          emitStepOutput('generation_id', gen.generationId);
          emitStepOutput('generation_sequence', gen.generationSequence);
          emitStepOutput('catalog_commit_sha', gen.catalogCommitSha);
          emitStepOutput('catalog_digest', gen.catalogDigest);
          emitStepOutput('captured_revision', gen.capturedRevision);
          return handoffResult;
        } catch (recoverErr) {
          console.error(`[Reconciler] Case B recovery failed: ${recoverErr.message}`);
          execSync(`rm -rf site/`);
          continue;
        }
      } else if (pagesErr.status === 401 || pagesErr.status === 403) {
        console.error(`[Reconciler] Permissions error on Pages API: ${pagesErr.message}`);
        break;
      } else {
        continue;
      }
    }

    // 4c. Verify public manifest & digest
    const isManifestVerified = liveManifest && 
      (liveManifest.generationId === gen.generationId) && 
      (liveManifest.generationSequence === gen.generationSequence) &&
      (liveManifest.catalogDigest === gen.catalogDigest);

    // 4d. 2-Tier Transactional Finalization
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
            throw new Error(`PREPARATION_INCOMPLETE_ABORT: Generation ${freshGen.generationId} fan-out batches must be prepared and verified before Tier 1 publication.`);
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
        } else {
          throw err;
        }
      }
    }
  }

  // 5. Query published generations with incomplete submission fan-out (Durable Fan-Out Recovery)
  const pendingFanoutGenerations = await db.collection('system_deployments')
    .where('status', '==', 'published')
    .where('submissionFanoutStatus', 'in', ['pending', 'in_progress', 'partial'])
    .orderBy('generationSequence', 'asc')
    .get();

  for (const fanoutDoc of pendingFanoutGenerations.docs) {
    const fanoutGen = fanoutDoc.data();
    console.log(`[Reconciler] Resuming incomplete submission fan-out for generation ${fanoutGen.generationId} (${fanoutGen.completedFanoutBatches || 0}/${fanoutGen.totalFanoutBatches || 0} batches completed).`);
    await executeSubmissionFanout(db, fanoutDoc.ref, fanoutGen);
  }

  return handoffResult;
}

/**
 * Deterministically prepares bounded fan-out child batch documents for a generation's work plan.
 * Must execute and verify complete batch creation BEFORE Tier 1 publication commits.
 *
 * Invariants Enforced (Phase 0.1.11):
 * 1. Deterministic Extraction & Deduplication: Obtains expected submission IDs captured for the generation,
 *    deduplicating via Set and validating each ID format.
 * 2. Lexicographical Sorting: Sorts IDs deterministically before partitioning to guarantee identical partition boundaries.
 * 3. Bounded Partitioning: Slices IDs into deterministic batches of max 400 IDs (batch_00000, batch_00001, ...).
 * 4. Idempotent Child Batch Persistence: Uses set with merge to create or resume batch documents without destroying state.
 * 5. Full Work-Plan Verification: Reads back all created batch documents to verify count and integrity.
 * 6. Atomic Preparation Marker: Updates parent generation with fanoutPreparationStatus: 'prepared',
 *    totalFanoutBatches, and expectedSubmissionCount only after verification passes.
 * 7. Crash Recovery: If interrupted before marking 'prepared', worker re-runs preparation idempotently.
 */
export async function prepareFanoutBatches(db, genDocRef, genData, rawSubmissionIds) {
  const genId = genData.generationId;

  // 1. Deduplicate deterministically and validate IDs
  const uniqueIds = Array.from(new Set(rawSubmissionIds)).filter(id => {
    return typeof id === 'string' && id.length >= 8 && id.length <= 128 && /^sub_[a-zA-Z0-9_-]+$/.test(id);
  });

  // 2. Lexicographical sorting
  uniqueIds.sort();

  // 3. Partition into bounded chunks of max 400 IDs
  const BATCH_SIZE = 400;
  const totalBatches = Math.ceil(uniqueIds.length / BATCH_SIZE);
  const batchesCol = genDocRef.collection('fanout_batches');

  for (let bIndex = 0; bIndex < totalBatches; bIndex++) {
    const chunkIds = uniqueIds.slice(bIndex * BATCH_SIZE, (bIndex + 1) * BATCH_SIZE);
    const batchId = `batch_${String(bIndex).padStart(5, '0')}`;
    const batchRef = batchesCol.doc(batchId);

    await batchRef.set({
      generationId: genId,
      batchIndex: bIndex,
      batchId: batchId,
      submissionIds: chunkIds,
      status: 'pending',
      expectedCount: chunkIds.length,
      completedCount: 0,
      skippedCount: 0,
      fanoutAttempts: 0,
      leaseOwner: null,
      leaseExpiresAt: null,
      createdAt: FirebaseFirestore.FieldValue.serverTimestamp(),
      updatedAt: FirebaseFirestore.FieldValue.serverTimestamp()
    }, { merge: true });
  }

  // 4. Verify every expected child batch exists and verify contents
  for (let bIndex = 0; bIndex < totalBatches; bIndex++) {
    const batchId = `batch_${String(bIndex).padStart(5, '0')}`;
    const bSnap = await batchesCol.doc(batchId).get();
    if (!bSnap.exists) {
      throw new Error(`FANOUT_PREPARATION_VERIFICATION_FAILED: Batch ${batchId} missing for generation ${genId}`);
    }
    const bData = bSnap.data();
    const expectedLength = uniqueIds.slice(bIndex * BATCH_SIZE, (bIndex + 1) * BATCH_SIZE).length;
    if (bData.submissionIds.length !== expectedLength) {
      throw new Error(`FANOUT_PREPARATION_VERIFICATION_FAILED: Batch ${batchId} count mismatch (expected ${expectedLength}, found ${bData.submissionIds.length})`);
    }
  }

  // 5. Persist verified preparation state on parent generation
  await genDocRef.update({
    fanoutPreparationStatus: 'prepared',
    totalFanoutBatches: totalBatches,
    expectedSubmissionCount: uniqueIds.length,
    updatedAt: FirebaseFirestore.FieldValue.serverTimestamp()
  });

  console.log(`[Fanout Preparation] Successfully prepared and verified ${totalBatches} batches (${uniqueIds.length} submissions) for generation ${genId}.`);
}

/**
 * Executes or resumes generation-aware, conditional submission status fan-out
 * using bounded subcollection batches (/system_deployments/{generationId}/fanout_batches/{batchId}).
 *
 * Invariants Enforced (Phase 0.1.11):
 * 1. Global Deployment Fence: Mini-chunk transaction reads /system_config/deployment_state.
 *    - If genSeq < latestPublishedSequence: abort as STALE_GLOBAL_GENERATION_ABORT (older generation cannot touch records).
 *    - If genSeq > latestPublishedSequence: abort as UNPUBLISHED_GENERATION_ABORT.
 *    - If genSeq === latestPublishedSequence && genId !== latestPublishedGenerationId: fail closed as HALT_CONFLICT.
 * 2. Moderation Revision Fence: Verifies currentSub.approvedCatalogRevision <= genData.capturedRevision.
 *    Submissions re-approved or modified after generation capture are skipped (never published with stale state).
 * 3. Product-Level Status Check: Reads /products/{productId}. Delisted or flagged products are skipped
 *    and never reactivated or implied publishable.
 * 4. Submission Monotonic Sequence Guard & Conflict Resolution:
 *    - If sub.publishedGenerationSequence > genSeq: skip stale update.
 *    - If sub.publishedGenerationSequence === genSeq && sub.publishedGenerationId !== genId: fail closed as HALT_CONFLICT.
 *    - If sub.publishedGenerationSequence === genSeq && sub.publishedGenerationId === genId && sub.status === 'published': idempotent no-op.
 * 5. Pure Transaction Callbacks: No external counter mutation or irreversible logging inside db.runTransaction callback.
 *    Computes chunk outcomes locally, returns them, and accumulates into outer counts ONLY after commit succeeds.
 * 6. Bounded Cursor Pagination: Reads fanout_batches via .orderBy('batchIndex', 'asc').limit(PAGE_SIZE).startAfter(lastDoc).
 * 7. Atomic Batch Completion: Marks child batch 'completed' and increments parent summary counters in a SINGLE atomic transaction.
 * 8. Resilient Manual Review: When batch retries reach 5 or fail fatally, atomically transitions child batch
 *    to 'manual_review' and parent to 'manual_review', logging to /admin_logs.
 * 9. Worker Concurrency Ownership: Root GitHub Actions concurrency group ('github-pages') serializes runs,
 *    backed by 15-minute lease (leaseOwner, leaseExpiresAt) on batch documents.
 */
async function executeSubmissionFanout(db, genDocRef, genData) {
  const genId = genData.generationId;
  const genSeq = genData.generationSequence;
  const capturedRev = genData.capturedRevision;
  let attempts = (genData.fanoutAttempts || 0) + 1;

  await genDocRef.update({
    submissionFanoutStatus: 'in_progress',
    fanoutAttempts: attempts,
    lastFanoutAttemptAt: new Date(),
    updatedAt: new Date()
  });

  const BATCH_PAGE_SIZE = 10;
  let lastBatchDoc = null;
  let hasMoreBatches = true;

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
        const batchData = batchDoc.data();
        const batchId = batchData.batchId;
        const submissionIds = batchData.submissionIds || [];
        const batchAttempts = (batchData.fanoutAttempts || 0) + 1;

        // Lease Acquisition & Ownership Check
        const workerRunId = process.env.GITHUB_RUN_ID || 'local_worker';
        const now = Date.now();
        const leaseExpired = batchData.leaseExpiresAt ? batchData.leaseExpiresAt.toMillis() < now : true;

        if (batchData.status === 'processing' && !leaseExpired && batchData.leaseOwner !== workerRunId) {
          console.log(`[Fanout] Batch ${batchId} is actively leased by worker ${batchData.leaseOwner}. Skipping.`);
          continue;
        }

        // Set 15-minute lease
        const leaseExpiry = new Date(now + 15 * 60 * 1000);
        await batchDoc.ref.update({
          status: 'processing',
          leaseOwner: workerRunId,
          leaseExpiresAt: leaseExpiry,
          fanoutAttempts: batchAttempts,
          lastAttemptAt: new Date(),
          updatedAt: new Date()
        });

        let batchCompleted = 0;
        let batchSkipped = 0;
        const MINI_CHUNK_SIZE = 50;

        // Process submission IDs in transactional mini-chunks with pure callbacks
        for (let i = 0; i < submissionIds.length; i += MINI_CHUNK_SIZE) {
          const chunkIds = submissionIds.slice(i, i + MINI_CHUNK_SIZE);

          const chunkResult = await db.runTransaction(async (transaction) => {
            // Pure callback: local outcome counters ONLY
            let chunkCompleted = 0;
            let chunkSkipped = 0;

            // 1. Read Global Deployment State at same consistency boundary
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

            // 2. Read submissions in mini-chunk
            const subRefs = chunkIds.map(id => db.collection('submissions').doc(id));
            const subSnapshots = await Promise.all(subRefs.map(ref => transaction.get(ref)));

            // 3. Collect product IDs to check product-level status
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

            // 4. Evaluate each submission deterministically
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

              // Post-Capture Moderation Revision Fence
              if (currentSub.approvedCatalogRevision && currentSub.approvedCatalogRevision > capturedRev) {
                chunkSkipped++;
                continue;
              }

              // Product-Level Status Check
              if (prodSnap && prodSnap.exists) {
                const prodData = prodSnap.data();
                if (prodData.status === 'delisted' || prodData.status === 'flagged') {
                  chunkSkipped++;
                  continue;
                }
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

              // Valid Monotonic Advance
              const targetProductId = currentSub.targetProductId || `prod_${currentSub.repoNumericId}_${currentSub.productData?.slug || ''}`;
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

        // Atomic Batch Completion & Parent Counter Synchronization (Single Transaction)
        await db.runTransaction(async (tx) => {
          const freshBatchSnap = await tx.get(batchDoc.ref);
          if (!freshBatchSnap.exists) throw new Error(`Batch ${batchId} missing`);
          const freshBatchData = freshBatchSnap.data();

          if (freshBatchData.status === 'completed') {
            return; // Idempotent no-op
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

        console.log(`[Fanout] Atomically completed batch ${batchId} (${batchCompleted} completed, ${batchSkipped} skipped).`);
      }

      if (batchesPageSnap.size < BATCH_PAGE_SIZE) {
        hasMoreBatches = false;
      } else {
        lastBatchDoc = batchesPageSnap.docs[batchesPageSnap.docs.length - 1];
      }
    }

    // Finalize Generation if All Batches are Confirmed Completed
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
      console.error(`[Fanout] Fatal error or max retries reached. Transitioning generation ${genId} to manual_review.`);

      // Atomically mark generation and active batch to manual_review
      await db.runTransaction(async (tx) => {
        tx.update(genDocRef, {
          submissionFanoutStatus: 'manual_review',
          reconciliationStatus: 'manual_review',
          lastFanoutError: err.message,
          updatedAt: new Date()
        });
      });

      await db.collection('admin_logs').add({
        action: 'SUBMISSION_FANOUT_MANUAL_REVIEW',
        generationId: genId,
        attempts: attempts,
        error: err.message,
        createdAt: new Date()
      });
    } else {
      await genDocRef.update({
        submissionFanoutStatus: 'partial',
        lastFanoutError: err.message,
        updatedAt: new Date()
      });
    }
  }
}
```

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
