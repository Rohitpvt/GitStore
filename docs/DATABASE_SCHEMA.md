# GitStore — Cloud Firestore Database Schema & Data Architecture

**Document Version:** 1.7.0  
**Phase:** Phase 0.1.6 — Step-Up Proof Integrity, Firestore Rules & Deployment Reconciliation  
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
4. **Action-Bound Step-Up Authentication Proof Integrity (Zero Raw OAuth Tokens):** Administrative commands and ownership claim requests write to restricted Firestore queues. Moderation requests strictly require fresh GitHub step-up reauthentication (`reauthenticateWithPopup`). To eliminate token exfiltration and persistent credential risks, **zero raw OAuth access tokens are stored in Firestore**. Instead, requests attach an action-bound cryptographic proof binding the caller's verified Firebase UID, immutable numeric GitHub ID (`request.auth.token.firebase.identities['github.com'][0]`), `auth_time`, action, target document path, and a single-use action nonce. The Security Rules enforce a strict 5-minute authentication freshness gate at creation time (`auth_time < 300s`), while worker execution operates on a 30-minute queue TTL from `createdAt`.
5. **12-State Recoverable Deployment State Machine & Recursive Canonical Catalog Digest:** Catalog deployments transition across a formal 12-state state machine. Manifest integrity is validated via a deterministic SHA-256 `catalogDigest` computed recursively over canonicalized catalog files (length-prefixed, null-byte framed `sha256:v1:...`, excluding manifest). Post-commit recovery inspects full Git history (`origin/main`) and paginated GitHub Commits API with branch ancestry verification (`git merge-base --is-ancestor`), handling missing Pages deployment IDs across Cases A–D via the official endpoint `GET /repos/{owner}/{repo}/pages/deployments/{catalogCommitSha}`.
6. **Durable Catalog Revisions & Transactional State Management:** Monotonic revision counters track requested vs published catalog state (`/system_config/catalog_state`). Inside an atomic Firestore transaction, the worker reads current catalog revisions, ensures monotonic advancement, and transactionally updates `isDirty = (currentRequested > newPublished)`, guaranteeing that changes arriving during compilation or deployment are never lost.
7. **Pre-Deployment Generation Ordering Guard:** Immediately prior to invoking `actions/deploy-pages@v5`, a dedicated guard evaluates the 4-case sequence matrix against Firestore `/system_config/deployment_state`, aborting stale or colliding deployments before they touch GitHub Pages.
8. **Decoupled Storefront Access:** Storefront reads bypass Firestore completely during normal user browsing, consuming static JSON compiled by GitHub Actions and served by GitHub Pages.

---

## 2. Collection Specifications

### 2.1 Collection: `users` (Strictly Private)
- **Path:** `/users/{uid}`
- **Document ID:** Firebase Auth UID (string).
- **Description:** Sensitive profile and authorization metadata for developers and platform administrators. Strictly private to the authenticated owner and batch worker.
- **Access Rule:** Read: Authenticated owner (`request.auth.uid == uid`) and worker via Admin SDK only. Zero public read access. Client write is restricted to safe profile fields via strict creation and update schemas (`diff().affectedKeys().hasOnly(...)`); client modification of privileged fields is blocked.

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

### 2.3 Collection: `submissions`
- **Path:** `/submissions/{submissionId}`
- **Document ID:** Globally unique ID: `sub_issue_{repoId}_{issueNumber}` (issue submissions) or `sub_dash_{nanoid}` (dashboard submissions).
- **Description:** Tracks every submitted application through its validation, review, and publication lifecycle.

```typescript
type SubmissionStatus = 
  | 'queued'               // Awaiting batch worker lease
  | 'processing'           // Actively leased and under validation by GitHub Actions
  | 'needs_review'         // Automated validation passed; awaiting manual admin approval
  | 'approved'             // Qualified for publication; queued for catalog compilation
  | 'catalog_generating'   // Leased by compilation batch
  | 'catalog_committed'    // JSON catalog & manifest committed to Git; awaiting deployment
  | 'deploying'            // GitHub Pages deployment workflow in progress
  | 'deployment_verified'  // Verified via public HTTP manifest and GitHub Deployments API
  | 'published'            // Confirmed live on GitHub Pages storefront
  | 'deployment_failed'    // Pages deploy failed/timed out; queued for recompilation retry
  | 'rejected'             // Failed manual review or policy inspection
  | 'failed';              // Automation error (e.g. 404 repo, corrupt release asset)

type OwnershipProofTier = 
  | 'direct_owner'         // Submitter is personal repo owner (eligible for auto-publish if verified)
  | 'org_member'           // Submitter is public org member (forced to needs_review)
  | 'committer'            // Submitter has merged commits (forced to needs_review)
  | 'community';           // Submitter has no proven affiliation (forced to needs_review)

interface SubmissionDocument {
  id: string;                        // "sub_issue_{repoId}_{issueNumber}" or "sub_dash_{nanoid}"
  developerUid: string | null;       // Foreign Key -> users/{uid} (null if submitted via issue and unclaimed)
  submitterGithubId: string | null;  // Numeric GitHub user ID e.g. "583231" (immutable)
  submitterGithubUsername: string | null;
  claimStatus: 'unclaimed' | 'claim_pending' | 'claimed';
  claimedAt: FirebaseFirestore.Timestamp | null;
  submissionSource: 'dashboard' | 'github_issue';
  issueNumber: number | null;

  // Globally Unique Entity Identifiers
  repoNumericId: number;             // GitHub's immutable numeric repo ID e.g. 892341901
  repoId: string;                    // "gh_repo_" + repoNumericId
  targetProductId: string;           // "prod_" + repoNumericId + "_" + productSlug

  // Product Payload
  productData: {
    name: string;                    // 3 - 60 chars
    slug: string;                    // URL-safe kebab-case string (e.g., "fast-terminal")
    repoUrl: string;                 // Full URL (https://github.com/owner/repo)
    repoOwner: string;               // Current owner handle (mutable on rename)
    repoName: string;                // Current repository name (mutable on rename)
    releaseTag: string;              // Specific tag (e.g., "v1.4.2")
    selectedAsset: {
      name: string;                  // Asset filename (e.g. "TerminalSetup.exe")
      extension: 'exe' | 'msi' | 'apk' | 'dmg' | 'zip' | 'tar.gz';
      sizeBytes: number;
      downloadUrl: string;
      assetId: number;               // GitHub numeric asset ID
    };
    shortDescription: string;
    fullDescription: string;
    category: string;
    iconUrl: string;
    screenshots: string[];
    supportedPlatforms: ('windows' | 'macos' | 'linux' | 'android' | 'web')[];
    version: string;
    license: string;
    liveDemoUrl: string | null;
  };

  // State Management & Processing Lease
  status: SubmissionStatus;
  statusReason: string | null;
  processingLease: {
    leaseHolder: string | null;
    leaseAcquiredAt: FirebaseFirestore.Timestamp | null;
    leaseExpiresAt: FirebaseFirestore.Timestamp | null;
  } | null;

  // Automated GitHub Verification Results
  validationResults: {
    repoExists: boolean;
    repoPublic: boolean;
    repoNotArchived: boolean;
    releaseExists: boolean;
    releaseNotDraft: boolean;
    assetExists: boolean;
    assetSizeMatch: boolean;
    validatedAt: FirebaseFirestore.Timestamp;
  } | null;

  // Repository Ownership Proof (Distinct from Verified Developer Badge)
  ownershipProof: {
    tier: OwnershipProofTier;
    evidence: string;                // e.g. "owner.id matches submitter.id" or "public org member"
    sufficientForAutoPublish: boolean; // True ONLY if tier == 'direct_owner'
    verifiedAt: FirebaseFirestore.Timestamp;
  } | null;

  adminNotes: string | null;
  reviewedBy: string | null;
  reviewedAt: FirebaseFirestore.Timestamp | null;
  publishedProductId: string | null;

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

### 2.5 Collection: `moderation_requests` (Action-Bound Proof, Zero Raw Tokens)
- **Path:** `/moderation_requests/{requestId}`
- **Document ID:** Auto-generated ID (`mod_req_{nanoid}`).
- **Description:** Decoupled asynchronous command queue enabling administrators to initiate privileged actions with mandatory fresh GitHub step-up reauthentication.
- **Proof Integrity & Zero Token Storage Invariant:** Clients authenticate interactively via `reauthenticateWithPopup(auth.currentUser, new GithubAuthProvider())`. The Firebase Auth ID token updates its `auth_time`. **No raw OAuth access tokens are written to Firestore.** The request stores an action-bound proof containing `providerId`, `authTime`, `actionNonce`, `targetDocumentPath`, and `actionHash` (SHA-256 over `${action}:${targetDocumentPath}:${actionNonce}:${authTime}`).
- **Dual Timing Model:**
  1. *Creation Gate (Security Rules):* Enforces `(request.time.toMillis() / 1000) - request.auth.token.auth_time < 300` and `auth_time <= (request.time.toMillis() / 1000) + 5`.
  2. *Worker Processing Window:* Document is completely immutable (`allow update, delete: if false;`). Worker processes requests within a 30-minute queue TTL (`now - doc.createdAt.toMillis() < 30 * 60 * 1000`).

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
  actionNonce: string;               // 64-char hex or UUIDv4 generated on client
  targetDocumentPath: string;        // e.g. "submissions/sub_issue_892341901_42"
  actionHash: string;                // SHA256(`${action}:${targetDocumentPath}:${actionNonce}:${authTime}`)
}

interface ModerationRequestDocument {
  id: string;
  action: ModerationAction;
  targetId: string;                  // Foreign key to submissions/{id}, users/{uid}, or products/{id}
  targetDocumentPath: string;        // Full path e.g. "submissions/sub_123"
  payload: {
    reason?: string;
    notes?: string;
    delistReason?: string;
  };
  requestedByUid: string;            // Firebase Auth UID (must match request.auth.uid)
  githubUserId: string;              // Immutable numeric GitHub user ID (must match request.auth.token.firebase.identities['github.com'][0])
  stepUpProof: StepUpProof;
  status: 'pending' | 'processing' | 'completed' | 'rejected' | 'expired';
  rejectionReason: string | null;    // Worker-managed
  processedByWorkerRunId: string | null;
  createdAt: FirebaseFirestore.Timestamp;
  processedAt: FirebaseFirestore.Timestamp | null;
}
```

### 2.6 Collection: `products`
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

### 2.7 Deployment Manifest & Recovery Metadata (`data/deployment-manifest.json`)
Compiled alongside `data/catalog.json` and deployed to GitHub Pages to provide public deployment verification. In accordance with non-circular integrity requirements, `data/deployment-manifest.json` does **not** contain the commit SHA of the Git commit containing it, records `generatedAt` rather than `deployedAt`, and includes `catalogDigest`:

```typescript
interface DeploymentManifest {
  generationId: string;              // "gen_{timestamp}_{randomSuffix}" e.g. "gen_1730300400000_a1b2c3d"
  generationSequence: number;        // Monotonically increasing sequence integer e.g. 104
  generatedAt: string;               // ISO 8601 UTC timestamp of catalog/manifest generation
  catalogDigest: string;             // "sha256:v1:{hex}" over canonicalized static files (excluding manifest)
  productCount: number;              // Total active products in catalog.json
  schemaVersion: "1.7.0";            // Schema specification version
  environment: 'github-pages';
}
```

### 2.8 Collection: `system_deployments` (12-State Durable Ledger)
- **Path:** `/system_deployments/{generationId}`
- **Document ID:** Generation ID matching manifest (`gen_{timestamp}_{randomSuffix}`).
- **Description:** Durable ledger of every catalog compilation and deployment attempt. Preserved permanently upon failure to prevent desynchronization between Git commits and Firestore records.
- **Client Access:** `allow read, write: if false;` (Internal collection; worker only via Admin SDK).

```typescript
type DeploymentSystemState =
  | 'approved'
  | 'catalog_generating'
  | 'catalog_committed'
  | 'deploying'
  | 'deployment_success'
  | 'deployment_verified'
  | 'published'
  | 'compilation_failed'
  | 'deployment_failed'
  | 'deployment_retained'
  | 'reconciliation_active'
  | 'stale_aborted';

type ReconciliationStatus = 
  | 'none'
  | 'sha_recovered'
  | 'pages_dispatched'
  | 'pending_retry'
  | 'reconciled'
  | 'abandoned'
  | 'stale_aborted';

interface SystemDeploymentRecord {
  generationId: string;              // Unique generation ID
  generationSequence: number;        // Monotonically incrementing sequence number
  catalogCommitSha: string | null;   // Git commit SHA (git rev-parse HEAD or recovered via git log blob inspection)
  catalogDigest: string;             // SHA-256 digest of canonicalized catalog files (sha256:v1:...)
  capturedRevision: number;          // Revision snapshot from system_config/catalog_state
  pagesDeploymentId: number | null;  // Dedicated GitHub Pages deployment ID (recovered via Cases A-D)
  expectedProductIds: string[];      // Array of submission IDs included in this generation
  status: DeploymentSystemState;
  reconciliationStatus: ReconciliationStatus;
  attemptHistory: Array<{
    attemptNumber: number;
    startedAt: FirebaseFirestore.Timestamp;
    completedAt?: FirebaseFirestore.Timestamp;
    stage: 'generation' | 'commit' | 'artifact_upload' | 'pages_deploy' | 'cdn_verification' | 'db_finalization';
    pagesDeploymentId?: number;
    pagesStatus?: 'succeed' | 'failed' | 'in_progress';
    error?: string;
  }>;
  lastObservedPagesStatus?: 'succeed' | 'failed' | 'in_progress';
  failureReason?: string;
  retryEligibility: boolean;
  createdAt: FirebaseFirestore.Timestamp;
  verifiedAt?: FirebaseFirestore.Timestamp;
  finalizedAt?: FirebaseFirestore.Timestamp;
}
```

### 2.9 Document: `system_config/deployment_state`
- **Path:** `/system_config/deployment_state`
- **Description:** Global monotonic deployment sequence pointer and concurrency lease tracker.
- **Client Access:** `allow read, write: if false;` (Internal collection; worker only via Admin SDK).

```typescript
interface GlobalDeploymentState {
  latestGenerationId: string;
  latestSequence: number;            // Current highest sequence number generated
  latestPublishedSequence: number;   // Current highest sequence confirmed published on Pages
  latestPublishedGenerationId: string;
  latestPublishedCommitSha: string;
  latestPublishedCatalogDigest: string;
  status: 'idle' | 'deploying';
  activeWorkerLease?: {
    workerId: string;
    leaseExpiresAt: FirebaseFirestore.Timestamp;
  };
  lastPublishedAt?: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.10 Document: `system_config/catalog_state` (Durable Revision Tracker)
- **Path:** `/system_config/catalog_state`
- **Description:** Durable monotonic revision tracking for work preservation across batch compilation cycles.
- **Client Access:** `allow read, write: if false;` (Internal collection; worker only via Admin SDK).

```typescript
interface GlobalCatalogState {
  latestRequestedRevision: number;    // Advanced on any approved submission, delisting, or category change
  latestPublishedRevision: number;    // Advanced when GitHub Pages deployment and CDN verification complete
  activeCapturedRevision: number;     // Revision snapshot captured by currently running compilation worker
  lastRevisionAdvancedAt: FirebaseFirestore.Timestamp;
  lastPublishedAt?: FirebaseFirestore.Timestamp;
  isDirty: boolean;                   // latestRequestedRevision > latestPublishedRevision
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.11 Collections: `categories`, `reports`, `admin_logs`, `system_config`
- `categories`: Taxonomy slugs (`/categories/{id}`). Public read, worker write.
- `reports`: User-generated abuse reports (`/reports/{id}`). Authenticated creation only (`request.auth != null`), strict schema bounds (`details.size() <= 2000`, `status == 'open'`), read/update/delete completely forbidden to clients.
- `admin_logs`: Immutable audit trail written strictly by Actions worker (`/admin_logs/{id}`). Client access forbidden (`allow read, write: if false;`).
- `system_config/global_settings`: Publicly readable platform configuration thresholds (`allow read: if true; allow write: if false;`).

---

## 3. Self-Healing Reconciliation Logic (`reconcilePublishedState`)

This routine runs at the start of every GitHub Actions worker execution to recover from reverse failure scenarios:
1. Pushed to `origin/main` but crashed before recording `catalogCommitSha`.
2. Pages deployment dispatched but `pagesDeploymentId` was not captured.
3. Pages deployment completed and live on CDN, but runner crashed before Firestore finalization.

### 3.1 Recursive Canonical JSON Serialization & Length-Prefixed Framing
Catalog digests must be invariant to JSON key order, whitespace, and operating system path separators, while being sensitive to array element order and data types:

```javascript
import { execSync } from 'child_process';
import crypto from 'crypto';
import fs from 'fs';
import path from 'path';

/**
 * Deeply canonicalizes any JavaScript value for deterministic hashing:
 * - Sorts object keys in lexicographical Unicode order recursively.
 * - Preserves exact array ordering while recursively canonicalizing elements.
 * - Formats numbers and primitives stably; rejects NaN / Infinity.
 * - Eliminates undefined, functions, and symbols.
 */
function canonicalizeJson(val) {
  if (val === null || typeof val !== 'object') {
    if (typeof val === 'number') {
      if (!Number.isFinite(val)) throw new TypeError('Non-finite number in catalog payload');
      return JSON.stringify(val);
    }
    return JSON.stringify(val);
  }
  if (Array.isArray(val)) {
    return '[' + val.map(item => canonicalizeJson(item)).join(',') + ']';
  }
  const keys = Object.keys(val).sort();
  return '{' + keys.map(k => JSON.stringify(k) + ':' + canonicalizeJson(val[k])).join(',') + '}';
}

/**
 * Computes deterministic SHA-256 digest over sorted static catalog files.
 * Uses unambiguous length-prefixed, null-byte framed file boundaries.
 * Format: "sha256:v1:" + hexDigest
 */
function computeCatalogDigest(baseDir) {
  const hash = crypto.createHash('sha256');
  const files = [];

  const mainCatalog = path.join(baseDir, 'data', 'catalog.json');
  const categoriesFile = path.join(baseDir, 'data', 'categories.json');
  if (fs.existsSync(mainCatalog)) files.push(mainCatalog);
  if (fs.existsSync(categoriesFile)) files.push(categoriesFile);

  const productsDir = path.join(baseDir, 'data', 'products');
  if (fs.existsSync(productsDir)) {
    const prodFiles = fs.readdirSync(productsDir)
      .filter(f => f.endsWith('.json'))
      .sort()
      .map(f => path.join(productsDir, f));
    files.push(...prodFiles);
  }

  // Sort files lexicographically by normalized POSIX relative path
  files.sort((a, b) => path.relative(baseDir, a).replace(/\\/g, '/').localeCompare(path.relative(baseDir, b).replace(/\\/g, '/')));

  for (const file of files) {
    const relPosixPath = path.relative(baseDir, file).replace(/\\/g, '/');
    const rawContent = fs.readFileSync(file, 'utf8');
    const parsed = JSON.parse(rawContent);
    const canonicalPayload = canonicalizeJson(parsed);
    const payloadBuffer = Buffer.from(canonicalPayload, 'utf8');

    // Unambiguous null-byte, length-prefixed framing: "FILE\0${relPath}\0${byteLength}\0"
    const header = Buffer.from(`FILE\0${relPosixPath}\0${payloadBuffer.length}\0`, 'utf8');
    hash.update(header);
    hash.update(payloadBuffer);
  }

  return 'sha256:v1:' + hash.digest('hex');
}
```

### 3.2 Executable Reconciler Logic

```javascript
/**
 * Reconciles live deployed GitHub Pages state, recovers unpersisted Git commit SHAs,
 * handles missing Pages deployment IDs across Cases A-D, and advances catalog revisions.
 */
async function reconcilePublishedState(db, octokit, pagesUrl) {
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
    if (res.ok) {
      liveManifest = await res.json();
    }
  } catch (err) {
    console.warn('[Reconciler] Could not reach public Pages manifest.', err.message);
  }

  // 3. Four-Case Monotonic Sequence & Idempotency Evaluation
  if (liveManifest) {
    const manifestSeq = liveManifest.generationSequence;
    const manifestId = liveManifest.generationId;

    if (manifestSeq < state.latestPublishedSequence) {
      console.warn(`[Reconciler] Case 1 (Stale): CDN manifest sequence (${manifestSeq}) is older than latest published (${state.latestPublishedSequence}). Aborting finalization.`);
      return;
    } else if (manifestSeq === state.latestPublishedSequence && manifestId === state.latestPublishedGenerationId) {
      console.log(`[Reconciler] Case 2 (Idempotent): CDN matches currently published generation ${manifestId} (seq ${manifestSeq}).`);
    } else if (manifestSeq === state.latestPublishedSequence && manifestId !== state.latestPublishedGenerationId) {
      console.error(`[Reconciler] CRITICAL SECURITY ALERT: Case 3 (Sequence Conflict). Manifest seq ${manifestSeq} claimed by distinct generation ${manifestId}, but Firestore has ${state.latestPublishedGenerationId}. Halting.`);
      await db.collection('admin_logs').add({
        action: 'DEPLOYMENT_SEQUENCE_CONFLICT',
        manifestId,
        storedId: state.latestPublishedGenerationId,
        sequence: manifestSeq,
        timestamp: new Date()
      });
      return; // Fail-closed
    } else {
      console.log(`[Reconciler] Case 4 (New Generation): CDN manifest seq ${manifestSeq} advances state past ${state.latestPublishedSequence}.`);
    }
  }

  // 4. Query active deployments pending finalization or retry across all non-terminal reconciliation states
  const pendingGenerations = await db.collection('system_deployments')
    .where('status', 'in', ['pending', 'built', 'dispatched', 'deploying', 'catalog_committed', 'deployment_retained'])
    .where('reconciliationStatus', 'in', ['none', 'sha_recovered', 'pages_dispatched', 'pending_retry'])
    .get();

  for (const genDoc of pendingGenerations.docs) {
    const gen = genDoc.data();

    // 4a. Full-History Git Search & Catalog Digest Verification for missing catalogCommitSha
    if (!gen.catalogCommitSha) {
      console.log(`[Reconciler] Generation ${gen.generationId} missing catalogCommitSha. Scanning origin/main history...`);
      try {
        const rawLog = execSync('git log --format="%H" origin/main -- data/deployment-manifest.json', { encoding: 'utf8' }).trim();
        const candidateShas = rawLog ? rawLog.split('\n') : [];
        let recoveredSha = null;

        for (const sha of candidateShas) {
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
          } catch (blobErr) { /* non-matching commit or not an ancestor */ }
        }

        // Fallback: Full Pagination Loop for GitHub Commits API
        if (!recoveredSha) {
          let page = 1;
          let hasMore = true;
          while (hasMore && page <= 5) {
            const apiCommits = await octokit.request('GET /repos/{owner}/{repo}/commits', {
              owner: 'Rohitpvt',
              repo: 'GitStore',
              path: 'data/deployment-manifest.json',
              sha: 'main',
              per_page: 100,
              page
            });
            if (!apiCommits.data || apiCommits.data.length === 0) break;

            for (const item of apiCommits.data) {
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
              } catch (apiErr) { /* continue */ }
            }
            if (recoveredSha) break;
            const linkHeader = apiCommits.headers['link'] || '';
            hasMore = linkHeader.includes('rel="next"');
            page++;
          }
        }

        if (recoveredSha) {
          console.log(`[Reconciler] Successfully recovered catalogCommitSha ${recoveredSha} for generation ${gen.generationId}.`);
          gen.catalogCommitSha = recoveredSha;
          await genDoc.ref.update({
            catalogCommitSha: recoveredSha,
            reconciliationStatus: 'sha_recovered',
            updatedAt: new Date()
          });
        } else {
          console.warn(`[Reconciler] No remote commit found on origin/main matching generation ${gen.generationId}. Concluding push never completed.`);
          await genDoc.ref.update({
            status: 'failed',
            failureReason: 'commit_not_found_on_remote',
            reconciliationStatus: 'abandoned',
            retryEligibility: false,
            updatedAt: new Date()
          });
          continue;
        }
      } catch (gitErr) {
        console.error(`[Reconciler] Git inspection failed during SHA recovery:`, gitErr.message);
        continue;
      }
    }

    // 4b. Missing Pages Deployment Status Handling via Official REST Endpoint
    let isPagesSucceed = false;
    let resolvedPagesDeploymentId = gen.pagesDeploymentId;

    try {
      // Official endpoint: GET /repos/{owner}/{repo}/pages/deployments/{catalogCommitSha}
      const lookupTarget = resolvedPagesDeploymentId || gen.catalogCommitSha;
      const pagesRes = await octokit.request('GET /repos/{owner}/{repo}/pages/deployments/{pages_deployment_id}', {
        owner: 'Rohitpvt',
        repo: 'GitStore',
        pages_deployment_id: lookupTarget
      });

      if (pagesRes.status === 200) {
        const deployData = pagesRes.data;
        resolvedPagesDeploymentId = deployData.id || resolvedPagesDeploymentId;
        if (deployData.status === 'built') {
          isPagesSucceed = true;
          await genDoc.ref.update({ pagesDeploymentId: resolvedPagesDeploymentId });
        } else if (deployData.status === 'queued' || deployData.status === 'in_progress') {
          console.log(`[Reconciler] Pages deployment for commit ${gen.catalogCommitSha} is currently ${deployData.status}.`);
          await genDoc.ref.update({ reconciliationStatus: 'pages_dispatched' });
          continue; // Non-terminal; wait for next cycle
        } else if (deployData.status === 'errored') {
          console.warn(`[Reconciler] Pages deployment errored for commit ${gen.catalogCommitSha}.`);
          await genDoc.ref.update({ reconciliationStatus: 'pending_retry' });
          continue;
        }
      }
    } catch (pagesErr) {
      if (pagesErr.status === 404) {
        // Case B: No remote Pages deployment exists for this commit SHA
        console.log(`[Reconciler] Case B: No Pages deployment found for commit ${gen.catalogCommitSha}. Executing recovery path.`);
        try {
          // 1. Re-checkout / restore git tree at catalogCommitSha
          execSync(`git checkout ${gen.catalogCommitSha}`);
          // 2. Validate catalog digest
          const computedDigest = computeCatalogDigest(process.cwd());
          if (computedDigest !== gen.catalogDigest) {
            console.error(`[Reconciler] Case B digest mismatch: computed ${computedDigest} !== stored ${gen.catalogDigest}. Abandoning.`);
            await genDoc.ref.update({
              status: 'failed',
              failureReason: 'digest_mismatch_during_recovery',
              reconciliationStatus: 'abandoned',
              updatedAt: new Date()
            });
            execSync(`git checkout main`);
            continue;
          }
          // 3. Reconstruct site/ staging directory
          execSync(`mkdir -p site/data site/assets`);
          execSync(`cp index.html admin.html dashboard.html submit.html product.html site/`);
          execSync(`cp -r assets/* site/assets/`);
          execSync(`cp -r data/* site/data/`);
          // 4. Mark workflow output to trigger deploy job
          console.log(`[Reconciler] Case B artifact reconstructed cleanly for commit ${gen.catalogCommitSha}.`);
          await genDoc.ref.update({
            reconciliationStatus: 'pages_dispatched',
            updatedAt: new Date()
          });
          execSync(`git checkout main`);
          continue;
        } catch (recoverErr) {
          console.error(`[Reconciler] Case B recovery failed:`, recoverErr.message);
          execSync(`git checkout main`);
          continue;
        }
      } else if (pagesErr.status === 401 || pagesErr.status === 403) {
        console.error(`[Reconciler] Permissions error checking Pages deployments:`, pagesErr.message);
        break; // Fatal credential issue; abort batch
      } else {
        console.warn(`[Reconciler] Transient Pages API check failure:`, pagesErr.message);
        continue;
      }
    }

    // 4c. Verify public manifest & catalog digest match this generation
    const isManifestVerified = liveManifest && 
      (liveManifest.generationId === gen.generationId) && 
      (liveManifest.generationSequence === gen.generationSequence) &&
      (liveManifest.catalogDigest === gen.catalogDigest);

    // 4d. Idempotent Finalization & Durable Revision Advancement inside Atomic Transaction
    if (isPagesSucceed && isManifestVerified) {
      console.log(`[Reconciler] Healing generation ${gen.generationId} (Commit ${gen.catalogCommitSha}). Finalizing Firestore to published.`);

      await db.runTransaction(async (transaction) => {
        // Read current catalog state INSIDE transaction
        const catalogStateRef = db.collection('system_config').doc('catalog_state');
        const liveCatalogStateDoc = await transaction.get(catalogStateRef);
        const liveCatalogState = liveCatalogStateDoc.exists ? liveCatalogStateDoc.data() : { latestRequestedRevision: 0, latestPublishedRevision: 0 };

        const capturedRev = gen.capturedRevision || gen.generationSequence;
        const currentPublishedRev = liveCatalogState.latestPublishedRevision || 0;
        const currentRequestedRev = liveCatalogState.latestRequestedRevision || capturedRev;

        // Monotonic assertion
        if (capturedRev <= currentPublishedRev && currentPublishedRev !== 0) {
          console.warn(`[Reconciler] Stale revision detected: capturedRev ${capturedRev} <= currentPublishedRev ${currentPublishedRev}. Skipping revision advance.`);
        } else {
          const isDirty = (currentRequestedRev > capturedRev);
          transaction.set(catalogStateRef, {
            latestPublishedRevision: capturedRev,
            isDirty,
            lastPublishedAt: new Date(),
            updatedAt: new Date()
          }, { merge: true });
        }

        // Finalize submissions
        for (const subId of gen.expectedProductIds) {
          const subRef = db.collection('submissions').doc(subId);
          transaction.update(subRef, {
            status: 'published',
            'processingLease.leaseHolder': null,
            updatedAt: new Date()
          });
        }

        // Finalize system deployment record
        transaction.update(genDoc.ref, {
          status: 'published',
          reconciliationStatus: 'reconciled',
          verifiedAt: new Date(),
          finalizedAt: new Date()
        });

        // Advance global published sequence monotonically
        transaction.set(db.collection('system_config').doc('deployment_state'), {
          latestPublishedSequence: gen.generationSequence,
          latestPublishedGenerationId: gen.generationId,
          latestPublishedCommitSha: gen.catalogCommitSha,
          latestPublishedCatalogDigest: gen.catalogDigest,
          status: 'idle',
          lastPublishedAt: new Date(),
          updatedAt: new Date()
        }, { merge: true });
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

    // Step-Up Freshness Gate: Requires GitHub provider authentication within 300s
    function isFreshlyAuthenticated() {
      return isAuthenticated() 
        && request.auth.token.firebase.sign_in_provider == 'github.com'
        && ('github.com' in request.auth.token.firebase.identities)
        && request.auth.token.auth_time != null
        && ((request.time.toMillis() / 1000) - request.auth.token.auth_time < 300)
        && (request.auth.token.auth_time <= (request.time.toMillis() / 1000) + 5);
    }

    // -------------------------------------------------------------------------
    // 1. Private User Documents (/users/{userId})
    // -------------------------------------------------------------------------
    // Strictly private to the user. Public access is forbidden.
    // Client cannot write or modify privileged fields (isAdmin, isVerifiedDeveloper,
    // reputationScore, githubNumericId).
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
    // Safe public projection. Publicly readable. Client write is restricted:
    // privileged fields (isVerifiedDeveloper, publishedProductIds) are worker-only.
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
          'id', 'developerUid', 'submitterGithubId', 'submitterGithubUsername',
          'claimStatus', 'claimedAt', 'submissionSource', 'issueNumber',
          'repoNumericId', 'repoId', 'targetProductId', 'productData',
          'status', 'statusReason', 'processingLease', 'validationResults',
          'ownershipProof', 'adminNotes', 'reviewedBy', 'reviewedAt',
          'publishedProductId', 'createdAt', 'updatedAt'
        ])
        && request.resource.data.id == submissionId
        && request.resource.data.developerUid == request.auth.uid
        && request.resource.data.status == 'queued'
        && request.resource.data.claimStatus == 'claimed'
        && request.resource.data.processingLease == null
        && request.resource.data.createdAt == request.time
        && request.resource.data.updatedAt == request.time;
      // Developers can edit metadata ONLY when queued or rejected; cannot tamper with ownership/lease/status
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
      allow update, delete: if false; // Privileged batch worker only via Admin SDK
    }

    // -------------------------------------------------------------------------
    // 5. Moderation Requests (/moderation_requests/{requestId})
    // -------------------------------------------------------------------------
    // Strictly requires fresh GitHub step-up reauthentication (< 300s). Zero raw tokens.
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
        && request.resource.data.targetId is string
        && request.resource.data.targetId.size() > 0
        && request.resource.data.targetDocumentPath is string
        && request.resource.data.targetDocumentPath.size() > 0
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
      allow update, delete: if false; // Privileged batch worker only via Admin SDK
    }

    // -------------------------------------------------------------------------
    // 6. Products (/products/{productId})
    // -------------------------------------------------------------------------
    match /products/{productId} {
      allow read: if resource.data.status == 'active';
      allow write: if false; // Worker only via Admin SDK
    }

    // -------------------------------------------------------------------------
    // 7. Categories (/categories/{categoryId})
    // -------------------------------------------------------------------------
    match /categories/{categoryId} {
      allow read: if true;
      allow write: if false;
    }

    // -------------------------------------------------------------------------
    // 8. Abuse Reports (/reports/{reportId})
    // -------------------------------------------------------------------------
    // Authenticated users only to prevent anonymous spam. Strict schema & size bounds.
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
      allow read, update, delete: if false; // Worker / Administrator only
    }

    // -------------------------------------------------------------------------
    // 9. Internal Collections & State (Default-Deny for Clients)
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
```
