# GitStore — System Design Document (SDD)

**Document Version:** 1.1.0  
**Phase:** Phase 0.1 — Architecture Audit, Hardening & Final Validation  
**Status:** Approved Technical Architecture (Hardened)  

---

## 1. High-Level Architecture Overview

GitStore is designed as a hybrid decoupled system utilizing **GitHub** as the static host, automation engine, issue tracker, and binary distribution channel, coupled with **Firebase** as the authoritative operational database, user identity service, and client-side security boundary.

```mermaid
flowchart TD
    subgraph ClientLayer["Client Layer (GitHub Pages CDN Fastly Backed)"]
        UserBrowser["Visitor / Consumer Browser"]
        DevBrowser["Developer Browser"]
        AdminBrowser["Admin Browser (Dashboard)"]
        StaticCatalog["Pre-Compiled Static JSON Catalogs\n(data/catalog.json, data/products/*.json)"]
    end

    subgraph FirebaseLayer["Platform 2: Firebase Managed Services (Spark Plan)"]
        FirebaseAuth["Firebase Authentication\n(GitHub OAuth + Magic Link)"]
        Firestore["Cloud Firestore Operational Database\n(users, submissions, moderation_requests,\nproducts, categories, reports, admin_logs)"]
        AppCheck["Firebase App Check\n(reCAPTCHA v3 Web Provider)"]
        SecurityRules["Firestore Security Rules\n(Granular RBAC & Immutability)"]
    end

    subgraph GitHubLayer["Platform 1: GitHub Core Services"]
        GHPages["GitHub Pages\n(Vanilla HTML5/CSS3/ES6 Storefront)"]
        GHIssues["GitHub Issue Forms\n(.github/ISSUE_TEMPLATE)"]
        GHReleases["Public GitHub Releases\n(External User Repositories)"]
        GHREST["GitHub REST API v3\n(Repo, Tag, Asset & Collaborator Validation)"]
        GHWorker["GitHub Actions Batch Worker\n(Scheduled every 10 min + Manual Dispatch)\n- Concurrency Group Lock\n- 15-Minute Leases\n- Two-Phase Compiler"]
    end

    %% Client Interactions
    UserBrowser -->|Read Static Catalog| StaticCatalog
    DevBrowser -->|Authenticate| FirebaseAuth
    DevBrowser -->|Write Submission status: queued| Firestore
    AdminBrowser -->|Enqueue Moderation Request| Firestore
    Firestore --- SecurityRules
    Firestore --- AppCheck

    %% Alternative Ingestion
    DevBrowser -.->|Open Issue Submission| GHIssues
    GHIssues -->|Worker Poll & Verify Ownership| GHWorker

    %% Batch Worker Engine
    GHWorker -->|1. Process Moderation Requests| Firestore
    GHWorker -->|2. Authorize Admin via Collaborator Check| GHREST
    GHWorker -->|3. Claim Leased Submissions Limit 3| Firestore
    GHWorker -->|4. Verify Repos & Assets| GHREST
    GHREST -->|Verify Source Existence & Commits| GHReleases
    GHWorker -->|5. Update Submissions status: approved| Firestore
    GHWorker -->|6. Compile Static JSON Catalogs| StaticCatalog
    GHWorker -->|7. Git Rebase, Commit & Push| GHPages
    GHWorker -->|8. Verify Deployment & Mark published| Firestore
```

---

## 2. Component Decomposition

### 2.1 Component 1: Static Web Frontend (`gh-pages`)
- **Technology:** HTML5, CSS3, Vanilla JavaScript (ES6 Modules). Zero frameworks or build dependencies.
- **Hosting:** GitHub Pages (Fastly edge network).
- **Key Responsibilities:**
  - High-performance, zero-latency catalog browsing using `fetch()` against pre-compiled `data/catalog.json`.
  - Client-side search, multi-faceted filtering (category, platform, license), and routing via URL hash or clean query parameters.
  - Interactive Developer Dashboard for managing submissions, displaying real-time submission status, and submitting applications.
  - Decoupled Admin Dashboard: Displays `needs_review` submissions and enqueues commands into `/moderation_requests` with real-time `onSnapshot` status tracking.
  - Integration with Firebase JavaScript SDK (v10+ Modular) for Auth, Firestore, and App Check.

### 2.2 Component 2: Cloud Firestore (Authoritative Data Store)
- **Role:** The single source of truth for dynamic transactional state, user profiles, pending submissions, moderation requests, system configurations, and audit logs.
- **Access Model:**
  - Public anonymous access is restricted to read-only on active products and categories.
  - Authenticated developers can create new submission documents and edit their own pending submissions.
  - Privileged fields (`isVerifiedDeveloper`, `isAdmin`, `status: published`, `processingLease`) are protected against client writes by strict Firestore Security Rules.
  - Administrative commands write exclusively to `/moderation_requests` with `status: 'pending'`.
  - Batch worker operations execute via privileged `firebase-admin` Node.js SDK inside GitHub Actions using encrypted secrets.

### 2.3 Component 3: GitHub Actions Batch Worker Engine
- **Workflow:** `.github/workflows/batch-processor.yml`.
- **Trigger:** Scheduled cron (`*/10 * * * *`) and `workflow_dispatch` (manual trigger).
- **Runtime:** `ubuntu-latest` running a modular Node.js runner script (`scripts/processor/`).
- **Core Operations:**
  1. **Concurrency Enforcement:** Configured with `concurrency: group: gitstore-worker, cancel-in-progress: false` to ensure atomic, non-overlapping processing runs.
  2. **Moderation Queue Execution:** Processes `/moderation_requests`, verifies requester's repository collaborator permission via GitHub API, executes privileged mutations using Admin SDK, and logs to `admin_logs`.
  3. **Issue Form Ingestion & Ownership Verification:** Ingests newly opened GitHub Issues tagged `submission:pending`, verifies ownership (owner/org/committer), assigns or provisions a Firebase UID, and registers the submission.
  4. **Lease Allocation:** Claims up to 3 `queued` or stale `processing` submissions by assigning a lease holder ID (`GITHUB_RUN_ID`) and 15-minute lease expiration.
  5. **GitHub REST API Verification:** Validates public repository status, release existence, asset name matching, size verification, and download URL fidelity.
  6. **Two-Phase Publication Protocol:** Sets qualified submissions to `approved`, compiles static JSON catalogs, commits to Git, verifies `actions/deploy-pages` success, and then updates status to `published`.

---

## 3. Data Flow & Lifecycles

### 3.1 Storefront Visitor Data Flow
```mermaid
sequenceDiagram
    autonumber
    actor Visitor as Visitor / Browser
    participant CDN as GitHub Pages CDN
    participant StaticData as data/catalog.json
    participant GHRel as External GitHub Releases

    Visitor->>CDN: Request index.html
    CDN-->>Visitor: Return Vanilla HTML/CSS/JS
    Visitor->>StaticData: fetch('data/catalog.json')
    StaticData-->>Visitor: Return Cached Products Array (~25KB)
    Note over Visitor: Local client-side search, filtering,<br/>and pagination run entirely in browser memory.<br/>Firestore reads consumed: EXACTLY ZERO.
    Visitor->>GHRel: Click 'Download (.exe/.dmg/.apk)'
    GHRel-->>Visitor: Stream binary directly from creator's GitHub Release asset
```

### 3.2 Submission to Publication Data Flow (Two-Phase Protocol)
```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer
    participant Client as GitStore Web UI
    participant Auth as Firebase Auth
    participant DB as Cloud Firestore
    participant Worker as GitHub Actions Worker
    participant GHAPI as GitHub REST API
    actor Admin as GitStore Admin

    Dev->>Auth: Sign in via GitHub OAuth / Magic Link
    Auth-->>Client: Return JWT & Firebase UID
    Dev->>Client: Fill Product Form & Submit
    Client->>DB: Create doc in submissions collection (status: 'queued')
    Note over Worker: Batch worker runs (cron or dispatch)
    Worker->>DB: Atomic Transaction Claim (batch limit = 3)
    Worker->>GHAPI: Verify repository, release tag, asset extension & size
    GHAPI-->>Worker: 200 OK (Metadata confirmed)
    alt Developer is UNVERIFIED or Community Nomination
        Worker->>DB: Update status to 'needs_review'
        Admin->>DB: Enqueue APPROVE in moderation_requests
        Worker->>GHAPI: Verify Admin repo collaborator permission
        GHAPI-->>Worker: 200 OK (permission: admin)
        Worker->>DB: Set submission status to 'approved'
    else Developer IS Verified AND Proven Owner
        Worker->>DB: Update status directly to 'approved'
    end
    Note over Worker: Phase 1: Compile & Deploy
    Worker->>Worker: Compile data/catalog.json & data/products/*.json
    Worker->>Worker: Git pull --rebase & push origin main
    Worker->>Worker: Dispatch actions/deploy-pages
    alt Deploy Succeeded
        Note over Worker: Phase 2: Finalize Database State
        Worker->>DB: Update status to 'published'
        Worker->>DB: Set /products/gh_{owner}_{repo} to 'active'
    else Deploy Failed
        Worker->>DB: Leave status in 'approved' (retried on next run)
    end
```

### 3.3 Privileged Administrator Workflow
```mermaid
sequenceDiagram
    autonumber
    actor Admin as Repository Maintainer
    participant Dashboard as Admin Dashboard (admin.html)
    participant FS as Cloud Firestore
    participant Actions as GitHub Actions Worker
    participant GH as GitHub REST API v3

    Admin->>Dashboard: Authenticate with GitHub OAuth
    Dashboard->>Dashboard: Render list of needs_review submissions
    Admin->>Dashboard: Click "Approve Submission"
    Dashboard->>FS: Write /moderation_requests (status: 'pending')
    Note over Dashboard: Dashboard listens via onSnapshot
    Actions->>FS: Fetch pending moderation requests
    Actions->>GH: GET /repos/{owner}/{repo}/collaborators/{username}/permission
    GH-->>Actions: Return permission: 'admin'
    Actions->>FS: Update /submissions/{id} to 'approved'
    Actions->>FS: Create /products/gh_{owner}_{repo}
    Actions->>FS: Write immutable entry to /admin_logs
    Actions->>FS: Update /moderation_requests/{id} (status: 'completed')
    FS-->>Dashboard: onSnapshot fires with status: 'completed'
    Dashboard-->>Admin: Show green "Published Live" confirmation
```

---

## 4. Modular Directory & Codebase Structure

```text
GitStore/
├── .github/
│   ├── workflows/
│   │   ├── batch-processor.yml     # Cron batch worker (every 10m)
│   │   └── pages-deploy.yml        # Pages deployment workflow
│   └── ISSUE_TEMPLATE/
│       └── product-submission.yml  # Structured issue form
├── assets/
│   ├── css/
│   │   ├── variables.css           # Design tokens (obsidian, neons, tech-cuts)
│   │   ├── base.css                # Reset, utility classes, blueprint grid
│   │   ├── components/
│   │   │   ├── buttons.css         # Chamfered buttons, hover states
│   │   │   ├── cards.css           # Product showcase cards, corner crosshairs
│   │   │   ├── navigation.css      # Fixed obsidian header, bracketed links
│   │   │   ├── badges.css          # Monospace status badges
│   │   │   └── forms.css           # Inputs, validation styling
│   │   └── pages/
│   │       ├── home.css            # Hero showcase, grid styling
│   │       ├── explore.css         # Filter drawer, search bar
│   │       ├── product-detail.css  # Detailed view, download action
│   │       ├── dashboard.css       # Submissions table & stats
│   │       └── admin.css           # Review queue & developer verification table
│   └── js/
│       ├── config/
│       │   └── firebase-config.js  # Public client keys & reCAPTCHA v3 init
│       ├── api/
│       │   ├── auth.js             # GitHub OAuth & Magic Link abstractions
│       │   ├── firestore.js        # Submissions submission, moderation enqueuer
│       │   └── catalog.js          # Fetching & caching static JSON catalogs
│       ├── components/
│       │   ├── navbar.js           # Responsive menu, user profile pill
│       │   ├── product-card.js     # Card rendering template
│       │   ├── filter-bar.js       # Category, platform, and license filters
│       │   └── toast.js            # Non-blocking terminal-style notifications
│       ├── utils/
│       │   ├── sanitizers.js       # Strict HTML & URL sanitization (DOMPurify)
│       │   ├── formatters.js       # Byte sizes, semver, dates
│       │   └── router.js           # Client-side hash/query parameter management
│       └── app.js                  # Global application bootstrapper
├── data/
│   ├── catalog.json                # Master searchable product index
│   ├── categories.json             # Category taxonomy and counts
│   └── products/                   # Granular product records
│       └── gh_{owner}_{repo}.json
├── scripts/
│   └── processor/
│       ├── index.js                # Main Actions runner entrypoint
│       ├── lease-manager.js        # Atomic transaction lease claimer
│       ├── issue-parser.js         # Issue form ingest & ownership validator
│       ├── moderation-handler.js   # Moderation request verifier & executor
│       ├── github-verifier.js      # REST API repository & asset validator
│       └── catalog-compiler.js     # Static JSON generator
├── docs/                           # Complete 13 Architectural Documents
├── index.html                      # Homepage
├── explore.html                    # Product Discovery & Search
├── product.html                    # Dynamic Product Detail Viewer
├── submit.html                     # Product Submission Wizard
├── dashboard.html                  # Developer Management Dashboard
├── admin.html                      # Administrator Moderation Portal
└── login.html                      # Authentication Page (OAuth & Magic Link)
```

---

## 5. Failure Recovery, Leases & Concurrency Protection

### 5.1 Concurrency Lock
GitHub Actions enforces concurrency at the workflow definition level:
```yaml
concurrency:
  group: gitstore-batch-worker
  cancel-in-progress: false
```
This guarantees that if an execution takes longer than 10 minutes, subsequent scheduled executions will queue rather than run concurrently, eliminating multi-runner race conditions.

### 5.2 Atomic Processing Leases & Recovery
- The batch processor queries for submissions where `status == 'queued' OR (status == 'processing' AND processingLease.leaseExpiresAt < NOW())`.
- It claims each item inside a Firestore transaction:
  ```javascript
  transaction.update(doc.ref, {
    status: 'processing',
    'processingLease.leaseHolder': process.env.GITHUB_RUN_ID,
    'processingLease.leaseExpiresAt': new Date(Date.now() + 15 * 60 * 1000)
  });
  ```
- If the runner crashes mid-execution, the lease expires after 15 minutes, allowing the subsequent worker run to reclaim and recover the submission safely.
- If deployment fails, submissions remain in `approved` and are re-compiled and re-deployed in the subsequent run.
