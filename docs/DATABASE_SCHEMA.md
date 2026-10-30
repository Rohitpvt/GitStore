# GitStore — Cloud Firestore Database Schema & Data Architecture

**Document Version:** 1.4.0  
**Phase:** Phase 0.1.3 — Deployment Integrity, Administrator Bootstrap & Workflow Security  
**Status:** Approved Schema Baseline (Hardened)  
**Database:** Google Cloud Firestore (Native Mode)

---

## 1. Overview & Data Modeling Principles

GitStore employs **Cloud Firestore** as its operational and transactional data backbone. The data model is optimized for document immutability, auditability, fine-grained client security rules, and minimal index overhead.

### Key Data Invariants
1. **Firebase Auth UID as Primary Application Identity:** All user-centric documents in `users` use the Firebase Auth UID as their document ID (`/users/{uid}`).
2. **Globally Unique Deterministic Entity IDs:** A single GitHub repository can distribute multiple independent software products. Repositories and products are identified by GitHub's immutable numeric repository ID:
   - Repository: `/repositories/gh_repo_{repoId}` (e.g. `gh_repo_892341901`)
   - Product: `/products/prod_{repoId}_{productSlug}` (e.g. `prod_892341901_viewer`)
   This mathematically prevents collisions even if two repositories under the same owner use the identical slug `viewer`.
3. **No Fake / Placeholder Firebase Accounts:** GitHub Issue submissions do **not** create placeholder Firebase Auth accounts. Unclaimed submissions record `submitterGithubId` (immutable numeric GitHub ID) and `claimStatus: 'unclaimed'`. They are claimed securely via an asynchronous `/claim_requests` queue when the developer authenticates via Firebase.
4. **Decoupled Privileged Queues (`moderation_requests`, `claim_requests`):** Administrative commands and ownership claim requests write to restricted Firestore queues. The Actions worker queries Firebase Auth Admin SDK to extract caller identities and authoritative permissions before applying updates.
5. **Decoupled Storefront Access:** Storefront reads bypass Firestore completely during normal user browsing, consuming static JSON compiled by GitHub Actions and served by GitHub Pages.

---

## 2. Collection Specifications

### 2.1 Collection: `users`
- **Path:** `/users/{uid}`
- **Document ID:** Firebase Auth UID (string).
- **Description:** Profile and authorization metadata for developers and platform administrators.

```typescript
interface UserDocument {
  uid: string;                       // Primary Key (matches Auth UID)
  email: string;                     // Verified contact email
  displayName: string;               // Display handle or GitHub full name
  avatarUrl: string;                 // URL to avatar
  githubUsername: string | null;     // Linked GitHub username (e.g. "octocat")
  githubNumericId: string | null;    // Verified immutable GitHub numeric ID (e.g. "583231")
  isVerifiedDeveloper: boolean;      // Admin-granted trust credential (distinct from repo ownership)
  isAdmin: boolean;                  // Informational flag (true authority verified via canonical repo collaborator API)
  reputationScore: number;
  submissionCounts: {
    total: number;
    approved: number;
    rejected: number;
  };
  createdAt: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.2 Collection: `submissions`
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

### 2.3 Collection: `claim_requests`
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

### 2.4 Collection: `moderation_requests`
- **Path:** `/moderation_requests/{requestId}`
- **Document ID:** Auto-generated ID (`mod_req_{nanoid}`).
- **Description:** Decoupled asynchronous command queue enabling administrators to initiate privileged actions from the frontend without browser-held service account credentials.

```typescript
type ModerationAction = 
  | 'APPROVE_SUBMISSION'
  | 'REJECT_SUBMISSION'
  | 'VERIFY_DEVELOPER'
  | 'REVOKE_DEVELOPER'
  | 'DELIST_PRODUCT';

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
  status: 'pending' | 'processing' | 'completed' | 'rejected';
  rejectionReason: string | null;    // e.g. "UNAUTHORIZED_CALLER: Not an admin/maintainer on GitStore canonical repository"
  processedByWorkerRunId: string | null;
  createdAt: FirebaseFirestore.Timestamp;
  processedAt: FirebaseFirestore.Timestamp | null;
}
```

### 2.5 Collection: `products`
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
  status: 'active' | 'delisted' | 'flagged';
  featured: boolean;
  publishedAt: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.6 Deployment Manifest & Recovery Metadata (`data/deployment-manifest.json`)
Compiled alongside `data/catalog.json` and deployed to GitHub Pages to provide public deployment verification. **Notice:** In accordance with non-circular integrity requirements, `data/deployment-manifest.json` does **not** contain the commit SHA of the Git commit containing it:

```typescript
interface DeploymentManifest {
  generationId: string;              // "gen_{timestamp}_{randomSuffix}" e.g. "gen_1730300400000_a1b2c3d"
  generationSequence: number;        // Monotonically increasing sequence integer e.g. 104
  deployedAt: string;                // ISO 8601 UTC timestamp
  productCount: number;              // Total active products in catalog.json
  schemaVersion: "1.4.0";            // Schema specification version
  environment: 'github-pages';
}
```

### 2.7 Collection: `system_deployments`
- **Path:** `/system_deployments/{generationId}`
- **Document ID:** Generation ID matching manifest (`gen_{timestamp}_{randomSuffix}`).
- **Description:** Durable ledger of every catalog compilation and deployment attempt. Preserved permanently upon failure to prevent desynchronization between Git commits and Firestore records.

```typescript
interface SystemDeploymentRecord {
  generationId: string;              // Unique generation ID
  generationSequence: number;        // Monotonically incrementing sequence number
  catalogCommitSha: string;          // Git commit SHA produced after commit (git rev-parse HEAD)
  expectedProductIds: string[];      // Array of submission IDs included in this generation
  status: 'deploying' | 'published' | 'failed' | 'stale';
  reconciliationStatus: 'pending_retry' | 'reconciled' | 'abandoned' | 'none';
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

### 2.8 Document: `system_config/deployment_state`
- **Path:** `/system_config/deployment_state`
- **Description:** Global monotonic deployment sequence pointer and concurrency lease tracker.

```typescript
interface GlobalDeploymentState {
  latestGenerationId: string;
  latestSequence: number;            // Current highest sequence number generated
  latestPublishedSequence: number;   // Current highest sequence confirmed published on Pages
  latestPublishedGenerationId: string;
  latestCommitSha: string;
  status: 'idle' | 'deploying';
  activeWorkerLease?: {
    workerId: string;
    leaseExpiresAt: FirebaseFirestore.Timestamp;
  };
  lastPublishedAt?: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.9 Collections: `categories`, `reports`, `admin_logs`, `system_config`
- `categories`: Taxonomy slugs (`/categories/{id}`).
- `reports`: User-generated abuse reports (`/reports/{id}`).
- `admin_logs`: Immutable audit trail written strictly by Actions worker (`/admin_logs/{id}`).
- `system_config`: Global thresholds (`/system_config/global_settings`).

---

## 3. Self-Healing Reconciliation Logic (`reconcilePublishedState`)

This routine runs at the start of every GitHub Actions worker execution to recover from reverse failure scenarios (e.g. GitHub Pages deployment succeeded and propagated to CDN, but worker crashed or network failed before updating Firestore). It checks the dedicated GitHub Pages deployments API and enforces monotonic sequencing:

```javascript
/**
 * Reconciles live deployed GitHub Pages state with Cloud Firestore.
 * Evaluates dedicated Pages deployment API and verifies public HTTP manifest.
 */
async function reconcilePublishedState(db, octokit, pagesUrl) {
  // 1. Fetch current global deployment state
  const stateDoc = await db.collection('system_config').doc('deployment_state').get();
  const state = stateDoc.exists ? stateDoc.data() : { latestPublishedSequence: 0 };

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

  // 3. Monotonic ordering assertion
  if (liveManifest && liveManifest.generationSequence < state.latestPublishedSequence) {
    console.warn(`[Reconciler] Stale manifest detected on CDN (seq ${liveManifest.generationSequence} < latest ${state.latestPublishedSequence}). Aborting finalization to prevent rollback.`);
    return;
  }

  // 4. Query active deployments pending finalization
  const pendingGenerations = await db.collection('system_deployments')
    .where('status', 'in', ['deploying', 'failed'])
    .where('reconciliationStatus', '==', 'pending_retry')
    .get();

  for (const genDoc of pendingGenerations.docs) {
    const gen = genDoc.data();

    // Check Pages deployment status using dedicated Pages deployment endpoint
    let isPagesSucceed = false;
    const latestAttempt = gen.attemptHistory?.[gen.attemptHistory.length - 1];
    if (latestAttempt?.pagesDeploymentId) {
      try {
        const pagesRes = await octokit.request('GET /repos/{owner}/{repo}/pages/deployments/{pages_deployment_id}', {
          owner: 'Rohitpvt',
          repo: 'GitStore',
          pages_deployment_id: latestAttempt.pagesDeploymentId
        });
        isPagesSucceed = (pagesRes.data.status === 'succeed');
      } catch (pagesErr) {
        console.warn(`[Reconciler] Pages deployment check failed for ID ${latestAttempt.pagesDeploymentId}`, pagesErr.message);
      }
    }

    // Verify public manifest matches this generation
    const isManifestVerified = liveManifest && 
      (liveManifest.generationId === gen.generationId) && 
      (liveManifest.generationSequence === gen.generationSequence);

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
          latestCommitSha: gen.catalogCommitSha,
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

    match /users/{userId} {
      allow read: if true;
      allow create: if isOwner(userId) 
        && request.resource.data.isAdmin == false
        && request.resource.data.isVerifiedDeveloper == false;
      allow update: if isOwner(userId)
        && request.resource.data.isAdmin == resource.data.isAdmin
        && request.resource.data.isVerifiedDeveloper == resource.data.isVerifiedDeveloper;
      allow delete: if false;
    }

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

    match /claim_requests/{requestId} {
      allow read: if isAuthenticated() && resource.data.requesterUid == request.auth.uid;
      // Client creation is strictly whitelisted: privileged fields (developerUid, verifiedGithubId, claimStatus, isAdmin, workerLease) are blocked
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

    match /moderation_requests/{requestId} {
      allow read: if isAuthenticated() && resource.data.requestedByUid == request.auth.uid;
      allow create: if isAuthenticated()
        && request.resource.data.keys().hasOnly(['requestId', 'action', 'targetId', 'payload', 'requestedByUid', 'status', 'createdAt'])
        && request.resource.data.requestId == requestId
        && request.resource.data.requestedByUid == request.auth.uid
        && request.resource.data.action in ['APPROVE_SUBMISSION', 'REJECT_SUBMISSION', 'VERIFY_DEVELOPER', 'REVOKE_DEVELOPER', 'DELIST_PRODUCT']
        && request.resource.data.targetId is string
        && request.resource.data.targetId.size() > 0
        && request.resource.data.status == 'pending'
        && request.resource.data.createdAt == request.time;
      allow update, delete: if false; // Privileged batch worker only via Admin SDK
    }

    match /products/{productId} {
      allow read: if resource.data.status == 'active';
      allow write: if false; // Worker only via Admin SDK
    }

    match /categories/{categoryId} {
      allow read: if true;
      allow write: if false;
    }

    match /reports/{reportId} {
      allow create: if request.resource.data.status == 'open';
      allow read, update, delete: if false;
    }

    match /admin_logs/{logId} { allow read, write: if false; }
    match /system_deployments/{generationId} { allow read: if true; allow write: if false; }
    match /system_config/{configId} { allow read: if true; allow write: if false; }
  }
}
```
