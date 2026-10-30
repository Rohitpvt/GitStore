# GitStore — Master Phase Plan & Development Control System

**Document Version:** 1.2.0  
**Last Updated:** 2026-10-30  
**Current Project State:** Phase 0.1.2 (Final Security & Architecture Corrections) — `IN_REVIEW` ⏳ | Awaiting User `APPROVED` Gate for Phase 1  
**Project Lead:** Technical Project Manager, Software Architect, & QA Lead  
**Governing Standard:** Strict Dual-Platform Confinement (**GitHub** & **Firebase** ONLY)

---

## 1. Project Phase Governance & Control System

### 1.1 Status Definitions & Governance Rules
To ensure strict quality control and eliminate premature execution:
- **`NOT_STARTED`**: Scope defined; dependencies not yet satisfied or phase not reached.
- **`IN_PROGRESS`**: Active development authorized by the user. Only one phase may be `IN_PROGRESS` at any given time.
- **`BLOCKED`**: Progress halted due to technical, architectural, security, or policy blockers.
- **`IN_REVIEW`**: Work and verification finished by Antigravity; presented to user for audit.
- **`COMPLETED`**: Antigravity has verified all deliverables, executed audits, and passed technical acceptance criteria.
- **`APPROVED`**: The user has formally audited the phase deliverables and explicitly authorized proceeding to the next phase.

> [!CRITICAL]
> **Cardinal Execution Rule:** A phase being marked `COMPLETED` or `IN_REVIEW` by Antigravity does **not** grant authorization to begin the next phase. Implementation work on Phase 1 requires explicit user `APPROVED` status.

---

## 2. Master Progress Summary

| Phase | Title | Status | Subphases Completed | Gate Authority |
|---|---|---|---|---|
| **Phase 0** | Architecture and Documentation | `APPROVED` ✅ | 100% (Baseline docs created) | User Approved |
| **Phase 0.1** | Architecture Hardening & Final Validation | `IN_REVIEW` ⏳ | 100% (Audited & Hardened) | User Review Pending |
| **Phase 0.1.2**| Final Security & Architecture Corrections | `IN_REVIEW` ⏳ | 100% (All 5 corrections closed) | User Review Pending |
| **Phase 1** | Frontend Core Foundation (Mock Storefront) | `NOT_STARTED` 🛑 | 0 / 5 Subphases | Requires Phase 0.1 Approval |
| **Phase 2** | Firebase Foundation and Authentication | `NOT_STARTED` 🛑 | 0 / 4 Subphases | Requires Phase 1 Approval |
| **Phase 3** | Developer Dashboard and Submission System | `NOT_STARTED` 🛑 | 0 / 4 Subphases | Requires Phase 2 Approval |
| **Phase 4** | GitHub Integration and Batch Processing | `NOT_STARTED` 🛑 | 0 / 4 Subphases | Requires Phase 3 Approval |
| **Phase 5** | Admin Moderation and Public Publishing | `NOT_STARTED` 🛑 | 0 / 4 Subphases | Requires Phase 4 Approval |
| **Phase 6** | Security, Integration & Quality Assurance | `NOT_STARTED` 🛑 | 0 / 4 Subphases | Requires Phase 5 Approval |
| **Phase 7** | Controlled Beta Launch | `NOT_STARTED` 🛑 | 0 / 4 Subphases | Requires Phase 6 Approval & Policy Gate |
| **Phase 8** | Post-Launch Improvements | `NOT_STARTED` 🛑 | Future Backlog | Post-Launch |

---

## 3. Comprehensive Phase Specifications

---

### Phase 0: Architecture and Documentation
- **Status:** `APPROVED` ✅ (Completed & Audited)
- **Objective:** Establish core product requirements, dual-platform architecture, database schemas, security models, and foundational documentation without writing application code.
- **Deliverables:** Complete 13 foundational documentation files.
- **Definition of Done (DoD):** All 13 documents authored, synchronized, zero application code written, verified against two-platform constraints.

---

### Phase 0.1: Architecture Hardening & Final Validation
- **Status:** `IN_REVIEW` ⏳ (Audited, Hardened, and Validated)
- **Objective:** Audit Phase 0 architecture for structural, security, and operational gaps; correct trust boundaries; establish canonical repository collaborator checks; eliminate placeholder accounts; implement multi-product repository hierarchy; design self-healing publication loop; remove unapproved platform assumptions (Fastly); correct reCAPTCHA quotas.

---

### Phase 0.1.2: Final Security and Architecture Corrections
- **Status:** `IN_REVIEW` ⏳ (Corrections Authored & Synchronized; Awaiting Gate)
- **Objective:** Execute independent corrections on 5 critical architecture domains:
  1. *Correction A — Collaborator Permission Mapping:* Fix `permission` vs `role_name` check against `GET /repos/Rohitpvt/GitStore/collaborators/{username}/permission`. Fail-closed policy for `admin` and `maintain`. Token access classified as pending integration verification.
  2. *Correction B — Provider Identity Lookup & Secure Claiming:* Use `.find(p => p.providerId === 'github.com')?.uid`. Implement asynchronous `/claim_requests` queue with Firebase Admin SDK validation and rules protection.
  3. *Correction C — Globally Unique Entity Identities:* Base repository and product keys on GitHub's immutable numeric repository ID (`gh_repo_{repoId}`, `prod_{repoId}_{productSlug}`). Eliminate cross-repo collisions.
  4. *Correction D — Two-Tier Publication State Machine:* Implement recoverable state machine (`approved` -> `catalog_committed` -> `deploying` -> `deployment_verified` -> `published`). Verify deployment via Deployments API and public HTTP `deployment-manifest.json`. Detail 10 failure recovery scenarios.
  5. *Correction E — App Check reCAPTCHA Enterprise:* Clarify that the 10,000 free monthly assessment allowance applies to reCAPTCHA Enterprise. Detail 1-hour token TTL caching and decouple assessments from Firestore operations.
  6. *Additional Safeguards & Ownership Distinction:* Enforce `process.env.GITHUB_REPOSITORY === 'Rohitpvt/GitStore'`, distinguish repo ownership vs release authority vs verified developer badge, and retain GitHub Pages policy gate as unresolved.
- **Files Modified:** All 13 architecture and specification documents under `docs/` and root `README.md`.
- **Definition of Done (DoD):** All 5 corrections and safeguards documented, test fixtures specified, zero application code written, synchronized with `ROADMAP.md`.

---

### Phase 1: Frontend Core Foundation
- **Status:** `NOT_STARTED` 🛑 (Awaiting User Authorization of Phase 0.1)
- **Objective:** Build the responsive, framework-free Vanilla HTML5/CSS3/ES6 static storefront hosted on GitHub Pages, loading pre-compiled mock catalogs.
- **Dependencies:** User approval of Phase 0.1.
- **Subphases:**
  - **1.1 — Design Tokens and Base Styling:** `variables.css`, `base.css`, component styles.
  - **1.2 — Shared UI Components:** Navigation header, product cards, crosshairs, platform badges, disclaimer banner, modals, toasts.
  - **1.3 — Static Catalog Engine:** Mock `catalog.json`, client utilities (`sanitizers.js`, `formatters.js`, `router.js`, `catalog.js`).
  - **1.4 — Storefront Pages:** `index.html`, `explore.html`, `product.html`.
  - **1.5 — Frontend Quality Audit:** Viewport responsiveness, WCAG 2.1 AA accessibility, Lighthouse 95+ scores.
- **Definition of Done (DoD):** All storefront views render cleanly with mock data, pass Lighthouse audits, and receive user sign-off.

---

### Phase 2: Firebase Foundation and Authentication
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Integrate Firebase Authentication, Cloud Firestore, and Firebase App Check without altering the static storefront CDN architecture.
- **Subphases:**
  - **2.1 — Firebase Project Integration:** Public client config, local emulator suite setup.
  - **2.2 — Authentication:** GitHub OAuth popup flow, Passwordless Email Magic Link flow, multi-provider account linking, session persistence, auth state listeners.
  - **2.3 — Firestore Foundation:** Document models, client profile synchronization, deploying hardened Security Rules and composite indexes.
  - **2.4 — App Check:** Web reCAPTCHA v3 provider configuration, local `DebugProvider` setup, lazy-loading on authenticated pages only.
- **Definition of Done (DoD):** Auth and DB foundations active and verified; emulator rules tests 100% pass.

---

### Phase 3: Developer Dashboard and Submission System
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Enable developers to submit software applications, inspect submission states, and manage their listings.
- **Subphases:**
  - **3.1 — Developer Dashboard (`dashboard.html`):** Profile summary, verified status badge, submission history table.
  - **3.2 — Product Submission Form (`submit.html`):** Multi-step wizard validating repo URL, release tag, asset dropdown, metadata, and license.
  - **3.3 — Firestore Submission Integration:** Client creation of `/submissions/{id}` documents with enforced `status: 'queued'`.
  - **3.4 — Submission Management:** Edit eligible draft/rejected submissions, display validation error details, prevent duplicate submissions.
- **Definition of Done (DoD):** Submissions write cleanly to Firestore with status `queued`; dashboard reflects real-time status.

---

### Phase 4: GitHub Integration and Batch Processing
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Build the automated verification, leasing, and static JSON compilation engine using GitHub Actions.
- **Subphases:**
  - **4.1 — GitHub API Integration:** REST API v3 verifier checking repo public status, release tags, asset existence, and sizes.
  - **4.2 — GitHub Issue Form Integration:** Issue template, issue ingestion parser, 4-tier ownership verification, submission claiming flow.
  - **4.3 — GitHub Actions Batch Worker:** Scheduled cron workflow, concurrency lock, 15-minute distributed leases, retry recovery.
  - **4.4 — Automated Moderation & Publication Decisions:** Multi-product key generation (`prod_{owner}_{slug}`), direct owner checks, routing unverified creators to `needs_review`.
- **Definition of Done (DoD):** End-to-end automated worker run successfully processes queued submissions.

---

### Phase 5: Admin Moderation and Public Catalog Publishing
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Implement the decoupled administrative moderation queue, developer verification, and Two-Phase catalog compilation.
- **Subphases:**
  - **5.1 — Administrator Dashboard (`admin.html`):** Pending review queue view, submission detail inspector, delist controls.
  - **5.2 — Secure Admin Operations:** Client enqueues commands into `/moderation_requests`; worker verifies repository collaborator permission on `Rohitpvt/GitStore` and applies updates via `firebase-admin`.
  - **5.3 — Catalog Generator:** Worker compiles active listings into `data/catalog.json`, `data/categories.json`, and `data/products/*.json`.
  - **5.4 — Two-Phase GitHub Pages Deployment:** Git commit & push with rebase, dispatch `actions/deploy-pages`, verify deployment success, finalize Firestore status to `published`.
- **Definition of Done (DoD):** Admin approval workflow live; catalog compiles and deploys to Pages automatically.

---

### Phase 6: Security, Integration & Quality Assurance
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Perform end-to-end integration testing, full security auditing, stress testing, and address the pre-launch policy gate.
- **Subphases:**
  - **6.1 — Security Audit:** Full penetration review of Security Rules, App Check, XSS vectors, and secret boundaries.
  - **6.2 — End-to-End Testing:** Complete automated test matrix from submission to public download.
  - **6.3 — Reliability & Failure Testing:** Self-healing reconciliation testing, simulated rate limits, lease contention.
  - **6.4 — Pre-Launch Policy Gate:** Submit formal request to GitHub Support to confirm compliance of GitStore's open software directory on GitHub Pages.
- **Definition of Done (DoD):** System certified secure, resilient, and policy-compliant.

---

### Phase 7: Controlled Beta Launch
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Deploy production environment and onboard initial cohort of open-source developers under live monitoring.
- **Definition of Done (DoD):** Live public operation under stable free quotas; user gives final launch sign-off.

---

### Phase 8: Post-Launch Improvements (Future Backlog)
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Evaluate and implement post-MVP enhancements strictly adhering to dual-platform constraints.

---

## 4. Change Control & Deviation Log

| Date | Phase | Deviation / Change | Architectural Justification | Approval Authority |
|---|---|---|---|---|
| 2026-10-30 | 0.1 | Added `/moderation_requests` command queue | Eliminates browser-held Admin SDK keys; routes admin authorization through GitHub repository collaborator checks. | Approved (ADR-05) |
| 2026-10-30 | 0.1.1 | Canonical GitStore Repository Trust Boundary | Fixes flaw in ADR-05: Collaborator checks verify admin permissions against `Rohitpvt/GitStore`, not the submitted product repo. Extracts verified GitHub numeric ID from Firebase Auth Admin SDK. | Approved (ADR-05) |
| 2026-10-30 | 0.1.1 | Removal of Fake Placeholder Firebase Accounts | Issue submissions record `submitterGithubId` and `claimStatus: 'unclaimed'`. Real claiming occurs upon OAuth login. Prevents auth DB pollution. | Approved (ADR-06) |
| 2026-10-30 | 0.1.1 | Strict Ownership Tiers (Direct Personal Owner Only) | Only personal repo owners qualify for auto-publication. Org members and committers forced to `needs_review` due to lack of release proof. | Approved (ADR-06) |
| 2026-10-30 | 0.1.1 | Self-Healing Reverse Failure Reconciliation | `reconcilePublishedState()` runs at start of worker cycle to heal unconfirmed Firestore documents if Pages deploy succeeded but DB update failed. | Approved (ADR-07) |
| 2026-10-30 | 0.1 | Pre-Launch Policy Gate for Pages | Replaces premature policy clearance with formal confirmation requirement before public launch. | Approved (ADR-08) |
| 2026-10-30 | 0.1.1 | Multi-Product Repository Decoupling | Decouples repos (`gh_repo_{owner}_{repo}`) from products (`prod_{owner}_{slug}`). Enables monorepos to offer multiple distinct tools. | Approved (ADR-09) |
| 2026-10-30 | 0.1.1 | Removal of Fastly Assumption & App Check Lazy-Loading | Removes Fastly references; relies strictly on GitHub Pages. Lazy-loads App Check on auth pages only to stay under 10k/month quota. | Approved (ADR-10) |
