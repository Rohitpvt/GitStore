# GitStore — Cloud Firestore Database Schema & Data Architecture

**Document Version:** 1.1.0  
**Phase:** Phase 0.1 — Architecture Audit, Hardening & Final Validation  
**Status:** Approved Schema Baseline (Hardened)  
**Database:** Google Cloud Firestore (Native Mode)

---

## 1. Overview & Data Modeling Principles

GitStore employs **Cloud Firestore** as its operational and transactional data backbone. The data model is optimized for document immutability, auditability, fine-grained client security rules, and minimal index overhead.

### Key Data Invariants
1. **Firebase Auth UID as Primary Key:** All user-centric documents in `users` use the Firebase Auth UID as their document ID (`/users/{uid}`).
2. **Deterministic Product Document IDs:** Active products use deterministic document keys: `/products/gh_{repoOwner}_{repoName}` (e.g., `/products/gh_torvalds_subsurface`). This guarantees write idempotency on retries and eliminates duplicate product records.
3. **Decoupled Privileged Actions Queue (`moderation_requests`):** To avoid exposing Admin SDK credentials to browser clients or requiring Cloud Functions, administrative commands are enqueued as immutable requests in `/moderation_requests` and executed by the GitHub Actions worker.
4. **Denormalized Snapshots:** Author and category information is selectively denormalized into `products` and `submissions` to eliminate cross-collection join queries in the client.
5. **No Direct Administrative Writes from Client:** Privileged fields (`isAdmin`, `isVerifiedDeveloper`, `processingLease`, `status`) cannot be written by clients; any attempt triggers immediate rejection at the Firestore Security Rules layer.
6. **Decoupled Storefront Access:** Storefront reads bypass Firestore completely during normal user browsing, consuming static JSON compiled by GitHub Actions. Firestore reads are reserved for dashboard operations and batch pipeline queries.

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
  avatarUrl: string;                 // URL to avatar (raw.githubusercontent.com / avatars.githubusercontent.com)
  githubUsername: string | null;     // Linked GitHub username (e.g. "octocat")
  githubUid: string | null;          // GitHub OAuth subject ID
  isVerifiedDeveloper: boolean;      // Admin-only toggle: enables auto-publication
  isAdmin: boolean;                  // Admin-only toggle: platform administrative permissions
  reputationScore: number;           // Calculated score based on approved releases and community feedback
  submissionCounts: {
    total: number;                   // Total submissions ever submitted
    approved: number;                // Successfully approved/published
    rejected: number;                // Rejected or flagged
  };
  createdAt: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.2 Collection: `submissions`
- **Path:** `/submissions/{submissionId}`
- **Document ID:** Auto-generated Firestore ID (`sub_xxxxxx`) or issue-derived key (`sub_issue_123`).
- **Description:** Tracks every submitted application through its validation, review, and publication lifecycle.

```typescript
type SubmissionStatus = 
  | 'queued'        // Awaiting batch worker lease
  | 'processing'    // Actively leased and under validation by GitHub Actions
  | 'needs_review'  // Automated validation passed; awaiting manual admin approval
  | 'approved'      // Approved by admin or auto-approved via verified developer status
  | 'published'     // Static files compiled and deployed to storefront
  | 'rejected'      // Failed manual review or policy inspection
  | 'failed';       // Automation error (e.g. 404 repo, corrupt release asset)

interface SubmissionDocument {
  id: string;                        // Document ID
  developerUid: string;              // Foreign Key -> users/{uid}
  submissionSource: 'dashboard' | 'github_issue';
  issueNumber: number | null;        // GitHub Issue number if ingested via issue form
  submissionType: 'author_submission' | 'community_nomination';

  // Product Payload
  productData: {
    name: string;                    // 3 - 60 chars
    slug: string;                    // URL-safe kebab-case string (e.g., "fast-terminal")
    repoUrl: string;                 // Full URL (https://github.com/owner/repo)
    repoOwner: string;               // Extracted owner handle
    repoName: string;                // Extracted repository name
    releaseTag: string;              // Specific tag (e.g., "v1.4.2")
    selectedAsset: {
      name: string;                  // Asset filename (e.g. "TerminalSetup.exe")
      extension: 'exe' | 'msi' | 'apk' | 'dmg' | 'zip' | 'tar.gz';
      sizeBytes: number;             // Asset size in bytes
      downloadUrl: string;           // Direct GitHub Releases browser_download_url
    };
    shortDescription: string;        // 10 - 160 chars summary
    fullDescription: string;         // Detailed markdown documentation (max 5000 chars)
    category: string;                // Foreign Key -> categories/{id}
    iconUrl: string;                 // Approved HTTPS URL to square application logo
    screenshots: string[];           // Array of 1 - 5 approved HTTPS image URLs
    supportedPlatforms: ('windows' | 'macos' | 'linux' | 'android' | 'web')[];
    version: string;                 // Semantic version string (e.g. "1.4.2")
    license: string;                 // Valid SPDX license expression (e.g., "MIT", "Apache-2.0")
    liveDemoUrl: string | null;      // Optional HTTPS preview URL
  };

  // State Management & Processing Lease
  status: SubmissionStatus;
  statusReason: string | null;       // Detailed human-readable explanation of status
  processingLease: {
    leaseHolder: string | null;      // GITHUB_RUN_ID or Worker ID
    leaseAcquiredAt: FirebaseFirestore.Timestamp | null;
    leaseExpiresAt: FirebaseFirestore.Timestamp | null; // 15-minute lease duration
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
    duplicateCheckPass: boolean;
    validatedAt: FirebaseFirestore.Timestamp;
  } | null;

  // Repository Ownership Proof (Audited in Phase 0.1)
  ownershipVerification: {
    checked: boolean;
    isOwner: boolean;
    isOrgMember: boolean;
    isCommitter: boolean;
    verifiedAt: FirebaseFirestore.Timestamp | null;
  } | null;

  // Moderation & Audit
  adminNotes: string | null;         // Internal feedback from administrator
  reviewedBy: string | null;         // Admin UID or Worker ID
  reviewedAt: FirebaseFirestore.Timestamp | null;
  publishedProductId: string | null; // Pointer to products/{id} once published

  createdAt: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.3 Collection: `moderation_requests` (New in Phase 0.1)
- **Path:** `/moderation_requests/{requestId}`
- **Document ID:** Auto-generated ID (`mod_req_xxxxxx`).
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
  requestedByGithubUsername: string; // GitHub username asserted by admin session
  status: 'pending' | 'processing' | 'completed' | 'rejected';
  rejectionReason: string | null;    // e.g. "UNAUTHORIZED_CALLER: Not a repository collaborator"
  processedByWorkerRunId: string | null;
  createdAt: FirebaseFirestore.Timestamp;
  processedAt: FirebaseFirestore.Timestamp | null;
}
```

### 2.4 Collection: `products`
- **Path:** `/products/{productId}`
- **Document ID:** Deterministic ID: `gh_{repoOwner}_{repoName}` (e.g. `gh_torvalds_subsurface`).
- **Description:** Authoritative live catalog of active and discoverable software applications.

```typescript
interface ProductDocument {
  id: string;                        // Deterministic: "gh_" + repoOwner + "_" + repoName
  slug: string;                      // Unique slug (e.g., "git-lens-pro")
  name: string;                      // Application name
  developerUid: string;              // Foreign Key -> users/{uid}
  developerSnapshot: {
    displayName: string;
    avatarUrl: string;
    githubUsername: string;
    isVerifiedDeveloper: boolean;
  };

  // Technical Metadata
  repoUrl: string;
  repoOwner: string;
  repoName: string;
  releaseTag: string;
  downloadAsset: {
    name: string;
    extension: string;
    sizeBytes: number;
    downloadUrl: string;             // Direct link to github.com/.../releases/download/...
    releasePublishedAt: FirebaseFirestore.Timestamp;
  };

  // Content
  shortDescription: string;
  fullDescription: string;
  category: string;                  // Ref -> categories/{id}
  iconUrl: string;
  screenshots: string[];
  supportedPlatforms: string[];
  version: string;
  license: string;
  liveDemoUrl: string | null;

  // Live Metrics (Periodically cached by batch worker)
  metrics: {
    githubStars: number;
    githubForks: number;
    releaseDownloads: number;
    gitStoreViews: number;
  };

  // Operational Flags
  status: 'active' | 'delisted' | 'flagged';
  featured: boolean;
  publishedAt: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.5 Collection: `categories`
- **Path:** `/categories/{categoryId}`
- **Document ID:** URL-safe slug (e.g. `desktop-apps`, `developer-tools`, `system-utilities`).
- **Description:** Taxonomy groupings for storefront filtering.

```typescript
interface CategoryDocument {
  id: string;                        // e.g., "developer-tools"
  name: string;                      // e.g., "Developer Utilities"
  description: string;               // Category overview
  iconName: string;                  // Design token icon reference
  displayOrder: number;              // Ordering in UI navigation
  activeProductCount: number;        // Denormalized count updated by batch worker
  createdAt: FirebaseFirestore.Timestamp;
}
```

### 2.6 Collection: `reports`
- **Path:** `/reports/{reportId}`
- **Document ID:** Auto-generated ID.
- **Description:** User-generated abuse, malware, broken link, and intellectual property reports.

```typescript
interface ReportDocument {
  id: string;
  productId: string;                 // Target product being reported
  reporterUid: string | null;        // Null if reported anonymously
  reason: 'malware' | 'broken_download' | 'copyright' | 'spam' | 'misleading' | 'other';
  details: string;                   // Up to 1000 chars of explanation
  status: 'open' | 'investigating' | 'resolved' | 'dismissed';
  resolutionNotes: string | null;
  reviewedBy: string | null;         // Admin UID or Worker Run ID
  createdAt: FirebaseFirestore.Timestamp;
  resolvedAt: FirebaseFirestore.Timestamp | null;
}
```

### 2.7 Collection: `admin_logs`
- **Path:** `/admin_logs/{logId}`
- **Document ID:** Auto-generated ID.
- **Description:** Immutable append-only audit trail recording every administrative decision and worker mutation.

```typescript
interface AdminLogDocument {
  id: string;
  operatorType: 'admin' | 'batch_worker';
  operatorUid: string;               // Admin UID or "SYSTEM_GITHUB_ACTIONS"
  operatorGithubUsername: string | null;
  action: 
    | 'SUBMISSION_APPROVE'
    | 'SUBMISSION_REJECT'
    | 'DEVELOPER_VERIFY'
    | 'DEVELOPER_REVOKE'
    | 'PRODUCT_DELIST'
    | 'REPORT_RESOLVE'
    | 'CONFIG_UPDATE'
    | 'SECURITY_UNAUTHORIZED_ADMIN_ATTEMPT';
  targetCollection: 'submissions' | 'products' | 'users' | 'reports' | 'system_config';
  targetId: string;
  diffPayload: Record<string, any>;  // Before/after state snapshot
  timestamp: FirebaseFirestore.Timestamp;
}
```

### 2.8 Collection: `system_config`
- **Path:** `/system_config/global_settings`
- **Document ID:** `global_settings`
- **Description:** Dynamic platform thresholds, batch worker parameters, and domain allowlists.

```typescript
interface SystemConfigDocument {
  maintenanceMode: boolean;
  maxSubmissionsPerUserPerDay: number; // Default: 3
  batchSize: number;                   // Default: 3 submissions per run
  leaseDurationMinutes: number;        // Default: 15 minutes
  autoPublishVerified: boolean;        // Default: true
  allowedAssetExtensions: string[];    // ['exe', 'msi', 'apk', 'dmg', 'zip', 'tar.gz']
  allowedImageDomains: string[];       // ['raw.githubusercontent.com', 'user-images.githubusercontent.com', 'avatars.githubusercontent.com']
  updatedAt: FirebaseFirestore.Timestamp;
}
```

---

## 3. Concrete Lease Claiming Pseudocode (Actions Worker)

The following transactional logic executes inside the GitHub Actions worker using `firebase-admin`:

```javascript
/**
 * Atomically claim up to 3 submissions using Firestore Transactions
 */
async function claimSubmissionBatch(db, workerRunId, batchLimit = 3) {
  const now = Date.now();
  const leaseDurationMs = 15 * 60 * 1000; // 15 minutes
  const leaseExpiresAt = new Date(now + leaseDurationMs);

  // 1. Query candidates (queued or expired lease)
  const candidateSnapshots = await db.collection('submissions')
    .where('status', 'in', ['queued', 'processing'])
    .orderBy('createdAt', 'asc')
    .limit(batchLimit * 2) // Fetch candidate buffer
    .get();

  const claimedSubmissions = [];

  for (const doc of candidateSnapshots.docs) {
    if (claimedSubmissions.length >= batchLimit) break;

    try {
      await db.runTransaction(async (transaction) => {
        const freshDoc = await transaction.get(doc.ref);
        if (!freshDoc.exists) return;

        const data = freshDoc.data();
        const isQueued = data.status === 'queued';
        const isExpiredLease = data.status === 'processing' &&
          data.processingLease &&
          data.processingLease.leaseExpiresAt.toMillis() < now;

        if (isQueued || isExpiredLease) {
          // Atomically acquire lease
          transaction.update(doc.ref, {
            status: 'processing',
            'processingLease.leaseHolder': workerRunId,
            'processingLease.leaseAcquiredAt': new Date(now),
            'processingLease.leaseExpiresAt': leaseExpiresAt,
            updatedAt: new Date(now)
          });

          claimedSubmissions.push({ id: doc.id, ...data });
        }
      });
    } catch (err) {
      console.warn(`[LeaseLock] Failed to claim submission ${doc.id}: ${err.message}`);
    }
  }

  return claimedSubmissions;
}
```

---

## 4. Hardened Firestore Security Rules Specification

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Helper functions
    function isAuthenticated() {
      return request.auth != null;
    }
    function isOwner(userId) {
      return isAuthenticated() && request.auth.uid == userId;
    }

    // Users Collection Rules
    match /users/{userId} {
      // Public read of developer profile
      allow read: if true;

      // Users can initialize their own profile document
      // Strictly enforces that role flags must be false on creation
      allow create: if isOwner(userId) 
        && request.resource.data.isAdmin == false
        && request.resource.data.isVerifiedDeveloper == false;

      // Users can update basic profile info, but CANNOT self-escalate roles
      allow update: if isOwner(userId)
        && request.resource.data.isAdmin == resource.data.isAdmin
        && request.resource.data.isVerifiedDeveloper == resource.data.isVerifiedDeveloper;

      // Deletions are forbidden from client
      allow delete: if false;
    }

    // Submissions Collection Rules
    match /submissions/{submissionId} {
      // Developers can read their own submissions
      allow read: if isAuthenticated() && resource.data.developerUid == request.auth.uid;
      
      // Developers can submit new products
      // Strictly enforces status 'queued' and null lease
      allow create: if isAuthenticated() 
        && request.resource.data.developerUid == request.auth.uid
        && request.resource.data.status == 'queued'
        && request.resource.data.processingLease == null
        && request.resource.data.publishedProductId == null;

      // Developers can only modify their own submissions if they are in queued or rejected state
      // Cannot touch processingLease, status cannot be forced to approved or published
      allow update: if isOwner(resource.data.developerUid)
        && (resource.data.status == 'queued' || resource.data.status == 'rejected')
        && request.resource.data.status == 'queued'
        && request.resource.data.developerUid == resource.data.developerUid
        && request.resource.data.processingLease == null;

      allow delete: if false;
    }

    // Moderation Requests Collection Rules (New in Phase 0.1)
    match /moderation_requests/{requestId} {
      // Requesting admin can read their own request status
      allow read: if isAuthenticated() && resource.data.requestedByUid == request.auth.uid;

      // Authenticated users can enqueue a moderation request
      // Must be created in 'pending' status; cannot modify or delete
      allow create: if isAuthenticated()
        && request.resource.data.requestedByUid == request.auth.uid
        && request.resource.data.status == 'pending';

      allow update, delete: if false;
    }

    // Products Collection Rules
    match /products/{productId} {
      // Public read for active products
      allow read: if resource.data.status == 'active';
      // ZERO client writes allowed. Only Actions worker via Admin SDK writes to products
      allow write: if false;
    }

    // Categories Collection Rules
    match /categories/{categoryId} {
      allow read: if true;
      allow write: if false;
    }

    // Reports Collection Rules
    match /reports/{reportId} {
      // Anyone can submit an abuse report with open status
      allow create: if request.resource.data.status == 'open';
      allow read, update, delete: if false;
    }

    // Admin Logs and System Config Rules
    match /admin_logs/{logId} {
      // ZERO client writes. Worker logs directly via Admin SDK
      allow read, write: if false;
    }
    match /system_config/{configId} {
      allow read: if true;
      allow write: if false;
    }
  }
}
```
