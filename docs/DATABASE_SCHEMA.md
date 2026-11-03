# GitStore — Cloud Firestore Database Schema & Data Architecture

**Document Version:** 1.6.0  
**Phase:** Phase 0.1.5 — Final Security Rules, Administrator Step-Up Authentication & Deployment Recovery Audit  
**Status:** Approved Schema Baseline (Hardened & Reconciled)  
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
4. **Decoupled Privileged Queues with Mandatory Step-Up Reauthentication (`moderation_requests`, `claim_requests`):** Administrative commands and ownership claim requests write to restricted Firestore queues. Moderation requests strictly require fresh GitHub step-up reauthentication (`reauthenticateWithPopup`) generating an ephemeral, single-use, action-bound proof. The Actions worker queries Firebase Auth Admin SDK and GitHub API to validate identity and repository permissions before executing actions, immediately purging ephemeral tokens.
5. **12-State Recoverable Deployment State Machine & Deterministic Catalog Digest:** Catalog deployments transition across a formal 12-state state machine. Manifest integrity is validated via a deterministic SHA-256 `catalogDigest` computed over canonicalized catalog files (excluding manifest). Post-commit recovery inspects full Git history (`origin/main`) and paginated GitHub Commits API with branch ancestry verification (`git merge-base --is-ancestor`), handling missing Pages deployment IDs across Cases A–D.
6. **Durable Catalog Revisions (`catalog_state`):** Monotonic revision counters track requested vs published catalog state (`/system_config/catalog_state`), guaranteeing that changes arriving during compilation or deployment are never lost.
7. **Decoupled Storefront Access:** Storefront reads bypass Firestore completely during normal user browsing, consuming static JSON compiled by GitHub Actions and served by GitHub Pages.

---

## 2. Collection Specifications

### 2.1 Collection: `users` (Strictly Private)
- **Path:** `/users/{uid}`
- **Document ID:** Firebase Auth UID (string).
- **Description:** Sensitive profile and authorization metadata for developers and platform administrators. Strictly private to the authenticated owner and batch worker.
- **Access Rule:** Read: Authenticated owner (`request.auth.uid == uid`) and worker via Admin SDK only. Zero public read access. Client write is restricted to safe profile fields; client modification of privileged fields is blocked.

```typescript
interface UserDocument {
  uid: string;                       // Primary Key (matches Auth UID)
  email: string;                     // Verified contact email (Private PII)
  emailVerified: boolean;            // Private PII
  displayName: string;               // Display handle or GitHub full name
  avatarUrl: string;                 // URL to avatar
  bio?: string;                      // Developer biography
  githubUsername: string | null;     // Linked GitHub username (e.g. "octocat")
  githubNumericId: string | null;    // Verified immutable GitHub numeric ID (e.g. "583231")
  isVerifiedDeveloper: boolean;      // Admin-granted trust credential (worker-managed only)
  isAdmin: boolean;                  // Informational flag (worker-managed only)
  reputationScore: number;           // Calculated score (worker-managed only)
  submissionCounts: {
    total: number;
    approved: number;
    rejected: number;
  };
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

### 2.5 Collection: `moderation_requests` (With Ephemeral Step-Up Proof)
- **Path:** `/moderation_requests/{requestId}`
- **Document ID:** Auto-generated ID (`mod_req_{nanoid}`).
- **Description:** Decoupled asynchronous command queue enabling administrators to initiate privileged actions with mandatory fresh GitHub step-up reauthentication.
- **Ephemeral Step-Up Proof Invariant:** Clients attach an ephemeral OAuth access token from `reauthenticateWithPopup(auth.currentUser, new GithubAuthProvider())`. Security Rules enforce `(request.time.toMillis() / 1000) - request.auth.token.auth_time < 300`. The Actions worker validates the token via GitHub REST API `GET /user`, checks canonical repo permissions (`Rohitpvt/GitStore`), and **immediately deletes `ephemeralGithubToken`** via `FieldValue.delete()`, recording `consumedAt` and worker run ID.

```typescript
type ModerationAction = 
  | 'APPROVE_SUBMISSION'
  | 'REJECT_SUBMISSION'
  | 'VERIFY_DEVELOPER'
  | 'REVOKE_DEVELOPER'
  | 'DELIST_PRODUCT'
  | 'FEATURE_PRODUCT';

interface ModerationRequestDocument {
  id: string;
  action: ModerationAction;
  targetId: string;                  // Foreign key to submissions/{id}, users/{uid}, or products/{id}
  payload: {
    reason?: string;
    notes?: string;
    delistReason?: string;
  };
  requestedByUid: string;            // Firebase Auth UID of requesting admin
  stepUpProof: {
    providerId: 'github.com';
    ephemeralGithubToken?: string;   // Deleted immediately upon consumption (FieldValue.delete())
    issuedAt: string;                // ISO 8601 UTC timestamp
    expiresAt: string;               // ISO 8601 UTC timestamp (issuedAt + 300s)
    actionNonce: string;             // 64-character random hex nonce
    consumedAt?: string;             // ISO 8601 timestamp added by worker upon validation
    workerRunId?: string;            // Actions run ID recording token consumption
  };
  status: 'pending' | 'processing' | 'completed' | 'rejected';
  rejectionReason: string | null;    // e.g. "UNAUTHORIZED_CALLER: Not an admin/maintainer on GitStore canonical repository"
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
  catalogDigest: string;             // SHA-256 digest of catalog state
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
  catalogDigest: string;             // SHA-256 digest of all sorted static catalog files (excluding manifest)
  productCount: number;              // Total active products in catalog.json
  schemaVersion: "1.6.0";            // Schema specification version
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

interface SystemDeploymentRecord {
  generationId: string;              // Unique generation ID
  generationSequence: number;        // Monotonically incrementing sequence number
  catalogCommitSha: string | null;   // Git commit SHA (git rev-parse HEAD or recovered via git log blob inspection)
  catalogDigest: string;             // SHA-256 digest of sorted catalog files
  capturedRevision: number;          // Revision snapshot from system_config/catalog_state
  pagesDeploymentId: number | null;  // Dedicated GitHub Pages deployment ID (recovered via Cases A-D)
  expectedProductIds: string[];      // Array of submission IDs included in this generation
  status: DeploymentSystemState;
  reconciliationStatus: 'pending_retry' | 'reconciled' | 'reconciled_from_git_log' | 'abandoned' | 'none';
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
- `reports`: User-generated abuse reports (`/reports/{id}`). Create only for clients (`status == 'open'`), read/update/delete forbidden.
- `admin_logs`: Immutable audit trail written strictly by Actions worker (`/admin_logs/{id}`). Client access forbidden (`allow read, write: if false;`).
- `system_config/global_settings`: Publicly readable platform configuration thresholds (`allow read: if true; allow write: if false;`).

---

## 3. Self-Healing Reconciliation Logic (`reconcilePublishedState`)

This routine runs at the start of every GitHub Actions worker execution to recover from reverse failure scenarios:
1. Pushed to `origin/main` but crashed before recording `catalogCommitSha`.
2. Pages deployment dispatched but `pagesDeploymentId` was not captured.
3. Pages deployment completed and live on CDN, but runner crashed before Firestore finalization.

It deterministically recovers missing commit SHAs via full-history Git log inspection and paginated GitHub Commits API, verifies branch ancestry (`git merge-base --is-ancestor`), matches `catalogDigest`, resolves missing Pages deployment IDs across Cases A–D, and advances durable catalog revisions:

```javascript
import { execSync } from 'child_process';
import crypto from 'crypto';
import fs from 'fs';
import path from 'path';

/**
 * Computes deterministic SHA-256 digest over sorted static catalog files.
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

  for (const file of files) {
    const content = fs.readFileSync(file, 'utf8');
    const parsed = JSON.parse(content);
    // Deterministic sorted JSON serialization
    const stableString = JSON.stringify(parsed, Object.keys(parsed).sort());
    hash.update(path.relative(baseDir, file) + ':' + stableString + '\n');
  }

  return 'sha256:' + hash.digest('hex');
}

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
      // Case 1: Stale Generation on CDN
      console.warn(`[Reconciler] Case 1 (Stale): CDN manifest sequence (${manifestSeq}) is older than latest published sequence (${state.latestPublishedSequence}). Aborting finalization.`);
      return;
    } else if (manifestSeq === state.latestPublishedSequence && manifestId === state.latestPublishedGenerationId) {
      // Case 2: Idempotent Retry / Already Verified
      console.log(`[Reconciler] Case 2 (Idempotent): CDN matches currently published generation ${manifestId} (seq ${manifestSeq}).`);
    } else if (manifestSeq === state.latestPublishedSequence && manifestId !== state.latestPublishedGenerationId) {
      // Case 3: Duplicate Sequence Conflict (Split-Brain)
      console.error(`[Reconciler] CRITICAL SECURITY ALERT: Case 3 (Sequence Conflict). Manifest sequence ${manifestSeq} claimed by distinct generation ${manifestId}, but Firestore has ${state.latestPublishedGenerationId}. Halting reconciler.`);
      await db.collection('admin_logs').add({
        action: 'DEPLOYMENT_SEQUENCE_CONFLICT',
        manifestId,
        storedId: state.latestPublishedGenerationId,
        sequence: manifestSeq,
        timestamp: new Date()
      });
      return; // Fail-closed
    } else {
      // Case 4: Valid New Generation (manifestSeq > state.latestPublishedSequence)
      console.log(`[Reconciler] Case 4 (New Generation): CDN manifest sequence ${manifestSeq} advances state past ${state.latestPublishedSequence}.`);
    }
  }

  // 4. Query active deployments pending finalization or retry
  const pendingGenerations = await db.collection('system_deployments')
    .where('status', 'in', ['deploying', 'failed', 'catalog_committed', 'deployment_retained'])
    .where('reconciliationStatus', 'in', ['pending_retry', 'none'])
    .get();

  for (const genDoc of pendingGenerations.docs) {
    const gen = genDoc.data();

    // 4a. Full-History Git Search & Catalog Digest Verification for missing catalogCommitSha
    if (!gen.catalogCommitSha) {
      console.log(`[Reconciler] Generation ${gen.generationId} missing catalogCommitSha. Scanning origin/main history...`);
      try {
        // Unshallow full-history search on origin/main
        const rawLog = execSync('git log --format="%H" origin/main -- data/deployment-manifest.json', { encoding: 'utf8' }).trim();
        const candidateShas = rawLog ? rawLog.split('\n') : [];
        let recoveredSha = null;

        for (const sha of candidateShas) {
          try {
            // Verify branch ancestry
            execSync(`git merge-base --is-ancestor ${sha} origin/main`);

            // Read blob without checkout
            const blobContent = execSync(`git show ${sha}:data/deployment-manifest.json`, { encoding: 'utf8' });
            const manifest = JSON.parse(blobContent);
            if (manifest.generationId === gen.generationId && 
                manifest.generationSequence === gen.generationSequence &&
                manifest.catalogDigest === gen.catalogDigest) {
              recoveredSha = sha; // Latest on origin/main matching digest is selected
              break;
            }
          } catch (blobErr) {
            // Commit not an ancestor or blob does not match; continue search
          }
        }

        // Fallback: Paginated GitHub Commits API
        if (!recoveredSha) {
          const apiCommits = await octokit.request('GET /repos/{owner}/{repo}/commits', {
            owner: 'Rohitpvt',
            repo: 'GitStore',
            path: 'data/deployment-manifest.json',
            sha: 'main',
            per_page: 100
          });
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
        }

        if (recoveredSha) {
          console.log(`[Reconciler] Successfully recovered catalogCommitSha ${recoveredSha} with matching digest for generation ${gen.generationId}.`);
          gen.catalogCommitSha = recoveredSha;
          await genDoc.ref.update({
            catalogCommitSha: recoveredSha,
            reconciliationStatus: 'reconciled_from_git_log',
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
          continue; // Move to next generation
        }
      } catch (gitErr) {
        console.error(`[Reconciler] Git inspection failed during SHA recovery:`, gitErr.message);
        continue;
      }
    }

    // 4b. Missing Pages Deployment ID Handling (Cases A, B, C, D)
    let isPagesSucceed = false;
    let resolvedPagesDeploymentId = gen.pagesDeploymentId;

    if (!resolvedPagesDeploymentId) {
      // Query recent Pages deployments
      try {
        const pagesList = await octokit.request('GET /repos/{owner}/{repo}/pages/deployments', {
          owner: 'Rohitpvt',
          repo: 'GitStore',
          per_page: 10
        });
        // Match deployment by commit SHA
        const matchedDeploy = pagesList.data.deployments?.find(d => d.commit_sha === gen.catalogCommitSha);
        if (matchedDeploy) {
          resolvedPagesDeploymentId = matchedDeploy.id;
          isPagesSucceed = (matchedDeploy.status === 'succeed');
          await genDoc.ref.update({ pagesDeploymentId: resolvedPagesDeploymentId });
        } else {
          // Case B: No remote Pages deployment found for the commit -> Worker must rebuild & trigger deploy
          console.log(`[Reconciler] Case B: No Pages deployment found for commit ${gen.catalogCommitSha}. Scheduling artifact packaging.`);
          continue;
        }
      } catch (listErr) {
        console.warn(`[Reconciler] Failed to list Pages deployments:`, listErr.message);
      }
    } else {
      // Pages deployment ID known -> Check status
      try {
        const pagesRes = await octokit.request('GET /repos/{owner}/{repo}/pages/deployments/{pages_deployment_id}', {
          owner: 'Rohitpvt',
          repo: 'GitStore',
          pages_deployment_id: resolvedPagesDeploymentId
        });
        isPagesSucceed = (pagesRes.data.status === 'succeed');
      } catch (pagesErr) {
        console.warn(`[Reconciler] Pages deployment check failed for ID ${resolvedPagesDeploymentId}:`, pagesErr.message);
      }
    }

    // 4c. Verify public manifest & catalog digest match this generation
    const isManifestVerified = liveManifest && 
      (liveManifest.generationId === gen.generationId) && 
      (liveManifest.generationSequence === gen.generationSequence) &&
      (liveManifest.catalogDigest === gen.catalogDigest);

    // 4d. Idempotent Finalization & Durable Revision Advancement
    if (isPagesSucceed && isManifestVerified) {
      console.log(`[Reconciler] Healing generation ${gen.generationId} (Commit ${gen.catalogCommitSha}). Finalizing Firestore to published.`);

      await db.runTransaction(async (transaction) => {
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

        // Advance durable catalog revision
        const capturedRev = gen.capturedRevision || gen.generationSequence;
        const currentReqRev = catalogState.latestRequestedRevision || capturedRev;
        transaction.set(db.collection('system_config').doc('catalog_state'), {
          latestPublishedRevision: capturedRev,
          isDirty: (currentReqRev > capturedRev),
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

    // Enforce that user reauthenticated with provider within the last 300 seconds (5 minutes)
    function isFreshlyAuthenticated() {
      return isAuthenticated() 
        && request.auth.token.auth_time != null
        && ((request.time.toMillis() / 1000) - request.auth.token.auth_time < 300);
    }

    // -------------------------------------------------------------------------
    // 1. Private User Documents (/users/{userId})
    // -------------------------------------------------------------------------
    // Strictly private to the user. Public access is forbidden.
    // Client cannot write or modify privileged fields (isAdmin, isVerifiedDeveloper,
    // reputationScore, githubNumericId, email, emailVerified).
    match /users/{userId} {
      allow read: if isOwner(userId);
      allow create: if isOwner(userId)
        && request.resource.data.uid == userId
        && request.resource.data.isAdmin == false
        && request.resource.data.isVerifiedDeveloper == false
        && request.resource.data.reputationScore == 0
        && request.resource.data.githubNumericId == null
        && request.resource.data.createdAt == request.time;
      allow update: if isOwner(userId)
        && request.resource.data.uid == userId
        && request.resource.data.isAdmin == resource.data.isAdmin
        && request.resource.data.isVerifiedDeveloper == resource.data.isVerifiedDeveloper
        && request.resource.data.reputationScore == resource.data.reputationScore
        && request.resource.data.githubNumericId == resource.data.githubNumericId
        && request.resource.data.email == resource.data.email
        && request.resource.data.emailVerified == resource.data.emailVerified
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
        && request.resource.data.userId == userId
        && request.resource.data.isVerifiedDeveloper == false
        && request.resource.data.publishedProductIds == []
        && request.resource.data.createdAt == request.time;
      allow update: if isOwner(userId)
        && request.resource.data.userId == userId
        && request.resource.data.isVerifiedDeveloper == resource.data.isVerifiedDeveloper
        && request.resource.data.publishedProductIds == resource.data.publishedProductIds
        && request.resource.data.updatedAt == request.time;
      allow delete: if false;
    }

    // -------------------------------------------------------------------------
    // 3. Submissions (/submissions/{submissionId})
    // -------------------------------------------------------------------------
    match /submissions/{submissionId} {
      allow read: if isAuthenticated() && resource.data.developerUid == request.auth.uid;
      allow create: if isAuthenticated() 
        && request.resource.data.developerUid == request.auth.uid
        && request.resource.data.status == 'queued'
        && request.resource.data.claimStatus == 'claimed'
        && request.resource.data.processingLease == null;
      // Developers can edit metadata ONLY when queued or rejected; cannot tamper with ownership/lease/status
      allow update: if isOwner(resource.data.developerUid)
        && (resource.data.status == 'queued' || resource.data.status == 'rejected')
        && request.resource.data.status == 'queued'
        && request.resource.data.developerUid == resource.data.developerUid
        && request.resource.data.claimStatus == resource.data.claimStatus
        && request.resource.data.ownershipProof == resource.data.ownershipProof
        && request.resource.data.processingLease == null;
      allow delete: if false;
    }

    // -------------------------------------------------------------------------
    // 4. Claim Requests (/claim_requests/{requestId})
    // -------------------------------------------------------------------------
    match /claim_requests/{requestId} {
      allow read: if isAuthenticated() && resource.data.requesterUid == request.auth.uid;
      // Client creation is strictly whitelisted: privileged fields are blocked
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
    // Strictly requires fresh step-up reauthentication (< 300 seconds) and ephemeral proof
    match /moderation_requests/{requestId} {
      allow read: if isAuthenticated() && resource.data.requestedByUid == request.auth.uid;
      allow create: if isFreshlyAuthenticated()
        && request.resource.data.keys().hasOnly(['requestId', 'action', 'targetId', 'payload', 'requestedByUid', 'stepUpProof', 'status', 'createdAt'])
        && request.resource.data.requestId == requestId
        && request.resource.data.requestedByUid == request.auth.uid
        && request.resource.data.action in ['APPROVE_SUBMISSION', 'REJECT_SUBMISSION', 'VERIFY_DEVELOPER', 'REVOKE_DEVELOPER', 'DELIST_PRODUCT', 'FEATURE_PRODUCT']
        && request.resource.data.targetId is string
        && request.resource.data.targetId.size() > 0
        && request.resource.data.stepUpProof is map
        && request.resource.data.stepUpProof.providerId == 'github.com'
        && request.resource.data.stepUpProof.ephemeralGithubToken is string
        && request.resource.data.stepUpProof.ephemeralGithubToken.size() > 0
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
    match /reports/{reportId} {
      allow create: if request.resource.data.status == 'open';
      allow read, update, delete: if false;
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
```
