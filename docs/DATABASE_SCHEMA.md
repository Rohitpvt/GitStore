# GitStore — Cloud Firestore Database Schema & Data Architecture

**Document Version:** 1.9.0  
**Phase:** Phase 0.1.8 — Node 24 Workflow Compatibility, Artifact Provenance & Transactional Deployment Integrity  
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
5. **Node 24 GitHub Actions Pipeline & Provenance-Preserving Staging:** Batch operations run on official GitHub Actions pinned to Node 24 releases (`actions/checkout@v5.0.0`, `actions/setup-node@v7.1.0`, `actions/upload-pages-artifact@v5.0.0`, `actions/deploy-pages@v5.0.1`). Artifact staging follows a single authoritative procedure: the `site/` staging directory is constructed exclusively from the verified source commit, its digest is recomputed and verified (`stagedDigest === catalogDigest`), and no subsequent workflow step copies from `main`. Step outputs are exported via direct append writes to `$GITHUB_OUTPUT`.
6. **12-State Deployment Recovery State Machine & Dedicated Pages Status (`succeed`):** Deployments transition across a formal 12-state state machine. Catalog integrity is proven via a deterministic SHA-256 `catalogDigest` computed recursively over canonicalized catalog files (length-prefixed, null-byte framed `sha256:v1:...`, excluding manifest). GitHub Pages deployment status is verified using the dedicated official endpoint `GET /repos/{owner}/{repo}/pages/deployments/{pages_deployment_id}` (where parameter accepts numeric deployment ID or commit SHA reference) checking for the documented success status `"status": "succeed"` (distinct from generic deployments and builds). In Case B recovery, the reconciler restores the Git tree, validates the digest, reconstructs `site/`, and explicitly exports `deployNeeded = true` to trigger artifact packaging and Pages deployment. Fallback commit recovery follows the full GitHub Commits API pagination loop without arbitrary page caps, distinguishes rate-limit or network errors (`INCOMPLETE_SEARCH`) from true missing commits (`EMPTY_SEARCH`), and validates commit SHA formats before shell execution.
7. **Transactional Generation Ordering & 2-Tier Finalization:** When finalizing deployment state, the worker executes an atomic transaction that reads current versions of `/system_config/catalog_state`, `/system_config/deployment_state`, and `/system_deployments/{generationId}` BEFORE any writes. Inside the transaction, it evaluates the shared ordering decision (`evaluateGenerationOrdering`) and catalog revision monotonicity (`capturedRevision >= latestPublishedRevision` and `capturedRevision <= latestRequestedRevision`). Stale generations abort immediately (`stale_aborted`) without modifying published sequence, published revision, commit SHA, or submission statuses. To strictly respect the Firestore 500-write limit, finalization uses a 2-tier architecture: the 3-write atomic transaction establishes the global canonical state, followed by idempotent batched writes (max 400 docs/batch) to fan out published statuses to submissions.
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
    repoUrl: string;                 // Must match https://github.com/owner/repo
    repoOwner: string;
    repoName: string;
    releaseTag: string;
    selectedAsset: {
      name: string;
      extension: string;
      sizeBytes: number;
      downloadUrl: string;
      assetId: number;
    };
    shortDescription: string;        // Max 200 chars
    fullDescription: string;         // Max 10000 chars (Markdown)
    category: string;
    iconUrl: string;
    screenshots: string[];           // Max 6 URLs
    supportedPlatforms: string[];    // At least 1 (windows, macos, linux, android, web)
    version: string;
    license: string;
    liveDemoUrl?: string | null;
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
  publishedProductId?: string | null; // Worker-Managed via Admin SDK
  publishedGenerationId?: string | null; // Worker-Managed via Admin SDK

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
- **Description:** Tracks compilation generations, commit provenance, staged digest validation, and deployment verification.
- **Client Access:** `allow read, write: if false;` (Internal collection; Admin SDK only).

```typescript
interface SystemDeploymentDocument {
  generationId: string;              // Primary key
  generationSequence: number;        // Strictly monotonic generation counter
  capturedRevision: number;          // Mandatory catalog revision snapshot (never substituted with sequence)
  finalizedRevision?: number | null; // Revision confirmed at publication finalization
  catalogCommitSha: string | null;   // 40-char git commit SHA on origin/main
  catalogDigest: string;             // Expected canonical digest (sha256:v1:...)
  stagedDigest?: string | null;      // Digest recomputed from staged site/ files
  pagesDeploymentReference: string | number | null; // Numeric ID or commit SHA reference
  status: 'pending' | 'catalog_committed' | 'deploying' | 'published' | 'failed' | 'stale_aborted' | 'deployment_retained';
  reconciliationStatus: 'none' | 'sha_recovered' | 'pages_dispatched' | 'pending_retry' | 'reconciled' | 'abandoned';
  lastObservedPagesStatus: 'succeed' | 'queued' | 'in_progress' | 'errored' | null;
  expectedProductIds: string[];
  attemptCount: number;
  createdAt: FirebaseFirestore.Timestamp;
  publishedAt?: FirebaseFirestore.Timestamp | null;
  finalizedAt?: FirebaseFirestore.Timestamp | null;
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

      // Fallback: GitHub Commits API with Full Pagination & Granular Error Classification
      if (!recoveredSha) {
        let currentUrl = `https://api.github.com/repos/Rohitpvt/GitStore/commits?sha=main&path=data/deployment-manifest.json&per_page=100`;
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
                const decoded = Buffer.from(fileContentRes.data.content, 'base64').toString('utf8');
                const manifest = JSON.parse(decoded);
                if (manifest.generationId === gen.generationId && 
                    manifest.generationSequence === gen.generationSequence &&
                    manifest.catalogDigest === gen.catalogDigest) {
                  recoveredSha = item.sha;
                  break;
                }
              } catch (blobErr) {
                if (blobErr.status === 404) {
                  // CANDIDATE_NO_MANIFEST: Commit did not touch manifest; continue to next candidate
                  continue;
                } else if (blobErr.status === 403 || blobErr.status === 429) {
                  // API_RATE_LIMIT: Search cannot complete; must not declare empty
                  console.warn(`[Reconciler] Rate limit hit during commit inspection for SHA ${item.sha}.`);
                  isSearchIncomplete = true;
                  break;
                } else if (blobErr.status >= 500) {
                  // API_SERVER_ERROR: Server error; search incomplete
                  console.warn(`[Reconciler] Upstream 5xx error during commit inspection for SHA ${item.sha}.`);
                  isSearchIncomplete = true;
                  break;
                } else {
                  console.warn(`[Reconciler] Error inspecting commit ${item.sha}: ${blobErr.message}`);
                  continue;
                }
              }
            }

            if (isSearchIncomplete || recoveredSha) break;

            const linkHeader = apiCommits.headers['link'];
            const nextMatch = linkHeader ? linkHeader.match(/<([^>]+)>;\s*rel="next"/) : null;
            currentUrl = nextMatch ? nextMatch[1] : null;
          } catch (pageErr) {
            console.warn(`[Reconciler] Commits API pagination failed: ${pageErr.message}`);
            isSearchIncomplete = true;
            break;
          }
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
      } else if (isSearchIncomplete) {
        console.warn(`[Reconciler] Search incomplete. Retaining generation ${gen.generationId} as pending_retry.`);
        await genDoc.ref.update({ reconciliationStatus: 'pending_retry', updatedAt: new Date() });
        continue;
      } else {
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
          
          const stagedDigest = computeCatalogDigest('site');
          if (stagedDigest !== gen.catalogDigest) {
            throw new Error(`STAGED_DIGEST_MISMATCH: staged ${stagedDigest} !== expected ${gen.catalogDigest}`);
          }

          await genDoc.ref.update({
            reconciliationStatus: 'pages_dispatched',
            stagedDigest: stagedDigest,
            updatedAt: new Date()
          });

          handoffResult = {
            deployNeeded: true,
            generationId: gen.generationId,
            generationSequence: gen.generationSequence,
            catalogCommitSha: gen.catalogCommitSha,
            catalogDigest: gen.catalogDigest,
            capturedRevision: gen.capturedRevision
          };
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

          // WRITE 3: Finalize system_deployments record
          transaction.update(genRef, {
            status: 'published',
            reconciliationStatus: 'reconciled',
            finalizedRevision: nextPublishedRev,
            verifiedAt: new Date(),
            finalizedAt: new Date(),
            updatedAt: new Date()
          });
        });

        // TIER 2: Idempotent Submission Status Fan-Out (Batched writes, max 400 docs/batch)
        const CHUNK_SIZE = 400;
        for (let i = 0; i < gen.expectedProductIds.length; i += CHUNK_SIZE) {
          const chunk = gen.expectedProductIds.slice(i, i + CHUNK_SIZE);
          const batch = db.batch();
          for (const subId of chunk) {
            const subRef = db.collection('submissions').doc(subId);
            batch.update(subRef, {
              status: 'published',
              publishedGenerationId: gen.generationId,
              'processingLease.leaseHolder': null,
              updatedAt: new Date()
            });
          }
          await batch.commit();
        }

      } catch (err) {
        if (err.message && (err.message.includes('STALE_GENERATION_ABORT') || err.message.includes('STALE_REVISION_ABORT'))) {
          console.warn(`[Reconciler] Stale finalization aborted safely. Marking generation ${gen.generationId} as stale_aborted.`);
          await genDoc.ref.update({ status: 'stale_aborted', reconciliationStatus: 'stale_aborted', updatedAt: new Date() });
        } else {
          throw err;
        }
      }
    }
  }

  return handoffResult;
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
        && request.resource.data.productData is map
        && request.resource.data.productData.keys().hasOnly([
          'name', 'slug', 'repoUrl', 'repoOwner', 'repoName', 'releaseTag',
          'selectedAsset', 'shortDescription', 'fullDescription', 'category',
          'iconUrl', 'screenshots', 'supportedPlatforms', 'version',
          'license', 'liveDemoUrl'
        ])
        && request.resource.data.productData.name is string
        && request.resource.data.productData.name.size() >= 3
        && request.resource.data.productData.name.size() <= 60
        && request.resource.data.productData.shortDescription is string
        && request.resource.data.productData.shortDescription.size() <= 200
        && request.resource.data.productData.fullDescription is string
        && request.resource.data.productData.fullDescription.size() <= 10000
        && request.resource.data.productData.screenshots is list
        && request.resource.data.productData.screenshots.size() <= 6
        && request.resource.data.productData.supportedPlatforms is list
        && request.resource.data.productData.supportedPlatforms.size() >= 1
        && request.resource.data.productData.selectedAsset is map
        && request.resource.data.productData.selectedAsset.keys().hasOnly(['name', 'extension', 'sizeBytes', 'downloadUrl', 'assetId'])
        && request.resource.data.productData.selectedAsset.sizeBytes is number
        && request.resource.data.productData.selectedAsset.sizeBytes > 0
        && request.resource.data.createdAt == request.time
        && request.resource.data.updatedAt == request.time;
      allow update: if isOwner(resource.data.developerUid)
        && (resource.data.status == 'queued' || resource.data.status == 'rejected')
        && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['productData', 'status', 'updatedAt'])
        && request.resource.data.status == 'queued'
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
