# GitStore — Cloud Firestore Database Schema & Data Architecture

**Document Version:** 1.2.0  
**Phase:** Phase 0.1.1 — Critical Architecture Corrections & Hardening  
**Status:** Approved Schema Baseline (Corrected)  
**Database:** Google Cloud Firestore (Native Mode)

---

## 1. Overview & Data Modeling Principles

GitStore employs **Cloud Firestore** as its operational and transactional data backbone. The data model is optimized for document immutability, auditability, fine-grained client security rules, and minimal index overhead.

### Key Data Invariants
1. **Firebase Auth UID as Primary Application Identity:** All user-centric documents in `users` use the Firebase Auth UID as their document ID (`/users/{uid}`).
2. **Multi-Product Decoupled Hierarchy:** A single GitHub repository can distribute multiple independent software products. Products use deterministic keys: `/products/prod_{repoOwner}_{productSlug}` (e.g., `prod_microsoft_procexp`, `prod_microsoft_autoruns`).
3. **No Fake / Placeholder Firebase Accounts:** GitHub Issue submissions do **not** create placeholder Firebase Auth accounts. Unclaimed submissions record `submitterGithubId` and `claimStatus: 'unclaimed'`. They are claimed when the developer authenticates via GitHub OAuth.
4. **Decoupled Privileged Actions Queue (`moderation_requests`):** Administrative commands write immutable requests to `/moderation_requests`. The Actions worker queries Firebase Auth Admin SDK to extract the caller's verified GitHub identity and validates repository permissions against GitStore's canonical repository.
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
  isVerifiedDeveloper: boolean;      // Admin-only toggle: enables auto-publication for direct owner repos
  isAdmin: boolean;                  // Informational flag (true authority comes from GitHub repo permissions)
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
- **Document ID:** Auto-generated Firestore ID (`sub_xxxxxx`) or issue key (`sub_issue_123`).
- **Description:** Tracks every submitted application through its validation, review, and publication lifecycle.

```typescript
type SubmissionStatus = 
  | 'queued'        // Awaiting batch worker lease
  | 'processing'    // Actively leased and under validation by GitHub Actions
  | 'needs_review'  // Automated validation passed; awaiting manual admin approval
  | 'approved'      // Qualified for publication; queued for catalog compilation
  | 'deploying'     // Static JSON compiled and pushed; awaiting GitHub Pages deployment confirmation
  | 'published'     // Confirmed live on GitHub Pages storefront
  | 'rejected'      // Failed manual review or policy inspection
  | 'failed';       // Automation error (e.g. 404 repo, corrupt release asset)

type OwnershipProofTier = 
  | 'direct_owner'        // Submitter is personal repo owner (eligible for auto-publish if verified)
  | 'org_member'          // Submitter is public org member (forced to needs_review)
  | 'committer'           // Submitter has merged commits (forced to needs_review)
  | 'community';          // Submitter has no proven affiliation (forced to needs_review)

interface SubmissionDocument {
  id: string;                        // Document ID
  developerUid: string | null;       // Foreign Key -> users/{uid} (null if submitted via issue and unclaimed)
  submitterGithubId: string | null;  // Numeric GitHub user ID (immutable)
  submitterGithubUsername: string | null;
  claimStatus: 'unclaimed' | 'claimed';
  submissionSource: 'dashboard' | 'github_issue';
  issueNumber: number | null;

  // Multi-Product & Repository Keys
  targetProductId: string;           // "prod_" + repoOwner + "_" + productSlug
  repoId: string;                    // "gh_repo_" + repoOwner + "_" + repoName

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
      sizeBytes: number;
      downloadUrl: string;
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

  // Repository Ownership Proof (Corrected in Phase 0.1.1)
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

### 2.3 Collection: `moderation_requests`
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
  status: 'pending' | 'processing' | 'completed' | 'rejected';
  rejectionReason: string | null;    // e.g. "UNAUTHORIZED_CALLER: Not a collaborator on GitStore canonical repository"
  processedByWorkerRunId: string | null;
  createdAt: FirebaseFirestore.Timestamp;
  processedAt: FirebaseFirestore.Timestamp | null;
}
```

### 2.4 Collection: `products`
- **Path:** `/products/{productId}`
- **Document ID:** Deterministic Key: `prod_{repoOwner}_{productSlug}` (e.g. `prod_microsoft_procexp`, `prod_microsoft_autoruns`).
- **Description:** Authoritative live catalog of active software products. Supports multiple products per repository.

```typescript
interface ProductDocument {
  id: string;                        // "prod_" + repoOwner + "_" + productSlug
  slug: string;                      // Unique slug (e.g., "procexp")
  name: string;                      // Application name
  repoId: string;                    // "gh_repo_" + repoOwner + "_" + repoName
  developerUid: string | null;       // Owner UID (null if unclaimed)
  developerSnapshot: {
    displayName: string;
    avatarUrl: string;
    githubUsername: string;
    isVerifiedDeveloper: boolean;
  };

  // Upstream Repository Metadata
  repoUrl: string;
  repoOwner: string;
  repoName: string;
  releaseTag: string;
  downloadAsset: {
    name: string;
    extension: string;
    sizeBytes: number;
    downloadUrl: string;
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

  status: 'active' | 'delisted' | 'flagged';
  featured: boolean;
  publishedAt: FirebaseFirestore.Timestamp;
  updatedAt: FirebaseFirestore.Timestamp;
}
```

### 2.5 Collections: `categories`, `reports`, `admin_logs`, `system_config`
- `categories`: Taxonomy slugs (`/categories/{id}`).
- `reports`: User-generated abuse reports (`/reports/{id}`).
- `admin_logs`: Immutable audit trail written strictly by Actions worker (`/admin_logs/{id}`).
- `system_config`: Global thresholds (`/system_config/global_settings`).

---

## 3. Self-Healing Reconciliation Pseudocode (`reconcilePublishedState`)

This routine runs at the start of every GitHub Actions worker execution to recover from reverse failure scenarios (e.g. GitHub Pages deployment succeeded, but Firestore finalization failed):

```javascript
/**
 * Reconciles live deployed catalog state with Cloud Firestore.
 * Ensures eventual consistency without distributed locks.
 */
async function reconcilePublishedState(db, deployedCatalog) {
  const publishedProductIds = new Set(deployedCatalog.map(p => p.id));

  // 1. Find any submissions that were deploying or approved whose product is already live
  const unconfirmedSnapshots = await db.collection('submissions')
    .where('status', 'in', ['approved', 'deploying'])
    .get();

  for (const doc of unconfirmedSnapshots.docs) {
    const sub = doc.data();
    if (publishedProductIds.has(sub.targetProductId)) {
      console.log(`[Reconciler] Healed submission ${doc.id}: already live in deployed catalog. Finalizing to published.`);
      
      await db.runTransaction(async (transaction) => {
        transaction.update(doc.ref, {
          status: 'published',
          publishedProductId: sub.targetProductId,
          'processingLease.leaseHolder': null,
          updatedAt: new Date()
        });

        const productRef = db.collection('products').doc(sub.targetProductId);
        transaction.set(productRef, {
          id: sub.targetProductId,
          slug: sub.productData.slug,
          name: sub.productData.name,
          repoId: sub.repoId,
          developerUid: sub.developerUid,
          status: 'active',
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
        && request.resource.data.processingLease == null;
      allow update: if isOwner(resource.data.developerUid)
        && (resource.data.status == 'queued' || resource.data.status == 'rejected')
        && request.resource.data.status == 'queued'
        && request.resource.data.developerUid == resource.data.developerUid
        && request.resource.data.processingLease == null;
      allow delete: if false;
    }

    match /moderation_requests/{requestId} {
      allow read: if isAuthenticated() && resource.data.requestedByUid == request.auth.uid;
      allow create: if isAuthenticated()
        && request.resource.data.requestedByUid == request.auth.uid
        && request.resource.data.status == 'pending';
      allow update, delete: if false;
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
    match /system_config/{configId} { allow read: if true; allow write: if false; }
  }
}
```
