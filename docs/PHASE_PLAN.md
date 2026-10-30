# GitStore — Master Phase Plan & Development Control System

**Document Version:** 1.0.0  
**Last Updated:** 2026-10-30  
**Current Project State:** Phase 0.1 (Architecture Hardening & Final Validation) — `COMPLETED` | Awaiting User `APPROVED` Gate for Phase 1  
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
> **Cardinal Execution Rule:** A phase being marked `COMPLETED` by Antigravity does **not** grant authorization to begin the next phase. Implementation work on Phase 1 requires explicit user `APPROVED` status.

---

## 2. Master Progress Summary

| Phase | Title | Status | Subphases Completed | Gate Authority |
|---|---|---|---|---|
| **Phase 0** | Architecture and Documentation | `APPROVED` ✅ | 100% (Baseline docs created) | User Approved |
| **Phase 0.1** | Architecture Hardening & Final Validation | `COMPLETED` ⏳ (Awaiting User Review) | 100% (All 7 critical audits closed) | User Review Pending |
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
- **Deliverables:**
  - Complete 13 foundational documentation files (`PRD.md`, `SDD.md`, `ARCHITECTURE.md`, `DATABASE_SCHEMA.md`, `AUTHENTICATION.md`, `SUBMISSION_PIPELINE.md`, `SECURITY.md`, `UI_DESIGN_SYSTEM.md`, `ROADMAP.md`, `DEPLOYMENT_GUIDE.md`, `TESTING_STRATEGY.md`, `COST_AND_LIMITS.md`, `README.md`).
- **Files Created / Modified:** Root and `docs/` documentation repositories.
- **Definition of Done (DoD):** All 13 documents authored, synchronized, zero application code written, verified against two-platform constraints.

---

### Phase 0.1: Architecture Hardening & Final Validation
- **Status:** `COMPLETED` ⏳ (Audit Finished; Awaiting User `APPROVED` Status)
- **Objective:** Audit Phase 0 architecture for structural, security, and operational gaps; resolve administrator execution model without Cloud Functions; establish GitHub Issue repository ownership proof; harden Firestore Security Rules and atomic leases; reassess GitHub Pages policy constraints; eliminate unrealistic free-tier claims.
- **Subphases:**
  1. *0.1.1 — Administrator Authorization Audit:* Decouple admin browser actions into an asynchronous `/moderation_requests` command queue verified against GitHub repository collaborator permissions (`admin` or `maintain`) via GitHub REST API.
  2. *0.1.2 — Issue Form Identity & Ownership Audit:* Establish multi-tier ownership framework (direct owner, public org member, committer, community nomination) and placeholder Firebase account linking.
  3. *0.1.3 — Security Rules & Lease Hardening:* Prevent client role manipulation (`isAdmin`, `isVerifiedDeveloper`) and client lease claiming; establish atomic transaction pseudocode.
  4. *0.1.4 — GitHub Pages Policy Reassessment:* Categorize policy compliance accurately; establish Phase 4 Pre-Launch Gate for written confirmation from GitHub Support.
  5. *0.1.5 — App Check & Spark Quotas Revalidation:* Replace reCAPTCHA Enterprise with reCAPTCHA v3 (1M calls free); document `RESOURCE_EXHAUSTED` failure boundaries; eliminate "free forever" guarantees.
  6. *0.1.6 — Batch Processing Reliability Audit:* Two-phase commit protocol to prevent database-catalog desynchronization; deterministic product document IDs (`gh_{owner}_{repo}`).
  7. *0.1.7 — Master Phase Tracking Plan:* Author `PHASE_PLAN.md` and synchronize with `ROADMAP.md`.
- **Files Created / Modified:**
  - Created: `PHASE_PLAN.md`
  - Modified (Updated to v1.1.0): `docs/PRD.md`, `docs/SDD.md`, `docs/ARCHITECTURE.md` (ADRs 05–10 added), `docs/DATABASE_SCHEMA.md`, `docs/AUTHENTICATION.md`, `docs/SUBMISSION_PIPELINE.md`, `docs/SECURITY.md`, `docs/UI_DESIGN_SYSTEM.md`, `docs/ROADMAP.md`, `docs/DEPLOYMENT_GUIDE.md`, `docs/TESTING_STRATEGY.md`, `docs/COST_AND_LIMITS.md`, `docs/README.md`, `README.md`.
- **Technical Acceptance Criteria:**
  - [x] Zero browser clients hold Firebase Admin SDK credentials.
  - [x] Unproven issue submitters cannot auto-publish.
  - [x] Two-phase commit protocol ensures catalog-database consistency.
  - [x] No third platform introduced.
- **Definition of Done (DoD):** All 7 audit domains resolved, 13 documentation files updated, `PHASE_PLAN.md` created, Phase 0.1 completion report delivered, zero application code written.

---

### Phase 1: Frontend Core Foundation
- **Status:** `NOT_STARTED` 🛑 (Awaiting User Authorization of Phase 0.1)
- **Objective:** Build the responsive, framework-free Vanilla HTML5/CSS3/ES6 static storefront hosted on GitHub Pages, loading pre-compiled mock catalogs.
- **Dependencies:** User approval of Phase 0.1.
- **Subphases:**
  - **1.1 — Design Tokens and Base Styling:**
    - Create `assets/css/variables.css` (obsidian `#040405`, neons `#00ff66`, `#05d9e8`, `#ff2a6d`, `#ffb800`, tech-cut clip-paths).
    - Create `assets/css/base.css` (reset, blueprint grid, responsive clamp typography).
    - Create `assets/css/components/` (`buttons.css`, `cards.css`, `badges.css`, `navigation.css`, `modals.css`, `forms.css`).
  - **1.2 — Shared UI Components:**
    - Fixed obsidian navigation header with character-bracketed links `[ EXPLORE ]`, `[ CATEGORIES ]`, `[ SUBMIT ]`.
    - Chamfered tech-cut product cards with corner crosshairs (`+`) and platform pills.
    - Status badges (`[ VERIFIED_DEV ]`, `[ QUEUED ]`, `[ NEEDS_REVIEW ]`).
    - Schematic non-execution legal disclaimer banner.
    - Accessible modal dialogs and terminal toast notifications.
  - **1.3 — Static Catalog Engine:**
    - Create mock datasets: `data/catalog.json`, `data/categories.json`, `data/products/*.json`.
    - Create modular client utilities: `assets/js/utils/sanitizers.js` (DOMPurify integration), `formatters.js` (bytes, dates, semver), `router.js` (hash/query state).
    - Create `assets/js/api/catalog.js`: In-memory caching, multi-facet filtering (category, platform, license), instant search.
  - **1.4 — Storefront Pages:**
    - `index.html`: Hero section, featured releases, category grid, live stats ticker.
    - `explore.html`: Multi-column catalog with sticky filter drawer and instant search.
    - `product.html`: Deep product showcase with screenshots, markdown documentation, platform pills, and direct release download CTA.
  - **1.5 — Frontend Quality Audit:**
    - Mobile, tablet, and desktop viewport responsiveness.
    - WCAG 2.1 AA contrast compliance and keyboard navigation.
    - Lighthouse audit targeting 95+ scores across Performance, Accessibility, Best Practices, and SEO.
- **Files Expected to be Created:**
  - `index.html`, `explore.html`, `product.html`
  - `assets/css/variables.css`, `assets/css/base.css`
  - `assets/css/components/*.css`
  - `assets/css/pages/*.css`
  - `assets/js/api/catalog.js`
  - `assets/js/components/*.js`
  - `assets/js/utils/*.js`
  - `data/catalog.json`, `data/categories.json`, `data/products/*.json`
- **Technical Acceptance Criteria:**
  - Storefront runs 100% on static files without server dependencies or frameworks.
  - Instant client-side search across 100+ mock products with zero perceptible latency.
  - Fully responsive across mobile (<768px), tablet (768–1023px), and desktop (>=1024px).
  - ZERO Firebase dependencies initialized in this phase.
- **Security Requirements:** Client-side markdown sanitized via DOMPurify before DOM injection.
- **Testing Requirements:** Playwright automated browser tests for catalog search, filtering, and responsive layout.
- **Definition of Done (DoD):** All storefront views render cleanly with mock data, pass Lighthouse audits, and receive user sign-off.

---

### Phase 2: Firebase Foundation and Authentication
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Integrate Firebase Authentication, Cloud Firestore, and Firebase App Check without altering the static storefront CDN architecture.
- **Dependencies:** Completed and approved Phase 1.
- **Subphases:**
  - **2.1 — Firebase Project Integration:** Public client config initialization (`assets/js/config/firebase-config.js`), local emulator suite configuration.
  - **2.2 — Authentication:** GitHub OAuth popup flow, Passwordless Email Magic Link flow, multi-provider account linking (`linkWithCredential`), session persistence (`browserLocalPersistence`), auth state listeners.
  - **2.3 — Firestore Foundation:** Document models, client profile synchronization, deploying hardened Security Rules (`firestore.rules`) and composite indexes (`firestore.indexes.json`).
  - **2.4 — App Check:** Web reCAPTCHA v3 provider configuration, local `DebugProvider` setup, origin enforcement.
- **Files Expected to be Created / Modified:**
  - `assets/js/config/firebase-config.js`
  - `assets/js/api/auth.js`
  - `assets/js/api/firestore.js`
  - `login.html`
  - `firestore.rules`, `firestore.indexes.json`
- **Technical Acceptance Criteria:**
  - Users authenticate via GitHub OAuth and Passwordless Email Links.
  - Zero password input fields exist on any page.
  - Security rules unit tests pass using `@firebase/rules-unit-testing` in local emulator.
  - Users are strictly prevented from escalating roles (`isAdmin`, `isVerifiedDeveloper`).
- **Security Requirements:** Zero exposure of service account keys; App Check reCAPTCHA v3 active.
- **Definition of Done (DoD):** Auth and DB foundations active and verified; emulator rules tests 100% pass.

---

### Phase 3: Developer Dashboard and Submission System
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Enable developers to submit software applications, inspect submission states, and manage their listings.
- **Dependencies:** Completed and approved Phase 2.
- **Subphases:**
  - **3.1 — Developer Dashboard (`dashboard.html`):** Developer profile summary, verified status badge, submission history table with reactive status indicators.
  - **3.2 — Product Submission Form (`submit.html`):** Multi-step wizard validating repo URL, release tag, asset dropdown, metadata, and license.
  - **3.3 — Firestore Submission Integration:** Client creation of `/submissions/{id}` documents with enforced `status: 'queued'` and `processingLease: null`.
  - **3.4 — Submission Management:** Edit eligible draft/rejected submissions, display validation error details, prevent duplicate submissions.
- **Files Expected to be Created / Modified:**
  - `submit.html`, `dashboard.html`
  - `assets/css/pages/dashboard.css`, `assets/css/pages/submit.css`
  - `assets/js/pages/dashboard.js`, `assets/js/pages/submit.js`
- **Technical Acceptance Criteria:**
  - Authenticated developers can submit valid open-source repositories.
  - Client-side validation enforces allowed asset extensions (`.exe`, `.msi`, `.apk`, `.dmg`, `.zip`, `.tar.gz`).
  - Users cannot approve or publish their own submissions.
- **Security Requirements:** Submission rate limiting (max 3/user/day) enforced via security rules/config.
- **Definition of Done (DoD):** Submissions write cleanly to Firestore with status `queued`; dashboard reflects real-time status.

---

### Phase 4: GitHub Integration and Batch Processing
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Build the automated verification, leasing, and static JSON compilation engine using GitHub Actions.
- **Dependencies:** Completed and approved Phase 3.
- **Subphases:**
  - **4.1 — GitHub API Integration:** REST API v3 verifier checking repo public status, release tags, asset existence, and sizes (`scripts/processor/github-verifier.js`).
  - **4.2 — GitHub Issue Form Integration:** Issue template (`.github/ISSUE_TEMPLATE/product-submission.yml`), issue ingestion parser, 3-tier ownership verification, placeholder account provisioning.
  - **4.3 — GitHub Actions Batch Worker:** Scheduled cron workflow (`.github/workflows/batch-processor.yml`), concurrency lock, 15-minute distributed leases, retry recovery.
  - **4.4 — Automated Moderation & Publication Decisions:** Verify developer status, detect duplicate products using deterministic keys (`gh_{owner}_{repo}`), route unverified creators to `needs_review`.
- **Files Expected to be Created / Modified:**
  - `.github/workflows/batch-processor.yml`
  - `.github/ISSUE_TEMPLATE/product-submission.yml`
  - `scripts/processor/*.js`
- **Technical Acceptance Criteria:**
  - Worker claims up to 3 submissions atomically using Firestore transactions.
  - Expired leases are safely recovered.
  - Non-owners submitting issues are categorized as `community_nomination` and gated to review.
- **Security Requirements:** All privileged operations run via `FIREBASE_SERVICE_ACCOUNT_JSON` secret; no untrusted binaries executed.
- **Definition of Done (DoD):** End-to-end automated worker run successfully processes queued submissions.

---

### Phase 5: Admin Moderation and Public Catalog Publishing
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Implement the decoupled administrative moderation queue, developer verification, and Two-Phase catalog compilation.
- **Dependencies:** Completed and approved Phase 4.
- **Subphases:**
  - **5.1 — Administrator Dashboard (`admin.html`):** Pending review queue view, submission detail inspector, delist controls.
  - **5.2 — Secure Admin Operations:** Client enqueues commands into `/moderation_requests`; worker verifies repository collaborator permission via GitHub API and applies updates via `firebase-admin`.
  - **5.3 — Catalog Generator:** Worker compiles active listings into `data/catalog.json`, `data/categories.json`, and `data/products/*.json`.
  - **5.4 — Two-Phase GitHub Pages Deployment:** Git commit & push with rebase, dispatch `actions/deploy-pages`, verify deployment success, finalize Firestore status to `published`.
- **Files Expected to be Created / Modified:**
  - `admin.html`, `assets/css/pages/admin.css`, `assets/js/pages/admin.js`
  - `scripts/processor/moderation-handler.js`
  - `scripts/processor/catalog-compiler.js`
  - `scripts/processor/finalize-publishing.js`
- **Technical Acceptance Criteria:**
  - Zero Admin SDK keys in browser.
  - Only authorized GitHub collaborators can execute moderation actions.
  - Static catalogs never desynchronize from Firestore on deployment failure.
- **Definition of Done (DoD):** Admin approval workflow live; catalog compiles and deploys to Pages automatically.

---

### Phase 6: Security, Integration & Quality Assurance
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Perform end-to-end integration testing, full security auditing, stress testing, and address the pre-launch policy gate.
- **Dependencies:** Completed and approved Phase 5.
- **Subphases:**
  - **6.1 — Security Audit:** Full penetration review of Security Rules, App Check, XSS vectors, and secret boundaries.
  - **6.2 — End-to-End Testing:** Complete automated test matrix from submission to public download.
  - **6.3 — Reliability & Failure Testing:** Simulated GitHub API rate limits, Firebase `RESOURCE_EXHAUSTED` boundaries, lease contention, deployment timeouts.
  - **6.4 — Pre-Launch Policy Gate:** Submit formal request to GitHub Support to confirm compliance of GitStore's open software directory on GitHub Pages.
- **Files Expected to be Created / Modified:**
  - `test/e2e/*.test.js`
  - `test/security/*.test.js`
  - Audit reports.
- **Technical Acceptance Criteria:**
  - Zero critical or high vulnerabilities.
  - 100% of test scenarios pass.
  - Pre-launch policy gate documented and resolved.
- **Definition of Done (DoD):** System certified secure, resilient, and policy-compliant.

---

### Phase 7: Controlled Beta Launch
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Deploy production environment and onboard initial cohort of open-source developers under live monitoring.
- **Dependencies:** Completed and approved Phase 6.
- **Subphases:**
  - **7.1 — Production Setup:** Production Firebase project configuration, GitHub Actions secrets injection.
  - **7.2 — Developer Beta Cohort:** Onboard 10–25 verified open-source maintainers; validate submission and download telemetry.
  - **7.3 — Operational Monitoring:** Monitor Spark read/write quotas, Fastly CDN bandwidth, and Actions runner minutes.
  - **7.4 — Public Launch:** Enable public storefront discovery.
- **Definition of Done (DoD):** Live public operation under stable free quotas; user gives final launch sign-off.

---

### Phase 8: Post-Launch Improvements (Future Backlog)
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Evaluate and implement post-MVP enhancements strictly adhering to dual-platform constraints.
- **Candidate Features:**
  - Star-based trending algorithm.
  - Community reviews and star ratings.
  - Automated GitHub Release webhook triggers.
  - RSS / Atom feeds for new application releases.
  - Category sharding for catalogs exceeding 1,000 products.

---

## 4. Change Control & Deviation Log

| Date | Phase | Deviation / Change | Architectural Justification | Approval Authority |
|---|---|---|---|---|
| 2026-10-30 | 0.1 | Added `/moderation_requests` command queue | Eliminates browser-held Admin SDK keys; routes admin authorization through GitHub repository collaborator checks. | Approved (ADR-05) |
| 2026-10-30 | 0.1 | Multi-tier Issue Ownership Framework | Prevents repository hijacking from issue submissions; gates unverified submitters to `needs_review`. | Approved (ADR-06) |
| 2026-10-30 | 0.1 | Two-Phase Publish Commit Protocol | Prevents catalog-database desynchronization if GitHub Pages deployment times out. | Approved (ADR-07) |
| 2026-10-30 | 0.1 | Pre-Launch Policy Gate for Pages | Replaces premature policy clearance with formal confirmation requirement before public launch. | Approved (ADR-08) |
| 2026-10-30 | 0.1 | Deterministic Product Keys (`gh_{owner}_{repo}`) | Guarantees batch write idempotency; eliminates duplicate product records on retry. | Approved (ADR-09) |
| 2026-10-30 | 0.1 | reCAPTCHA v3 & Spark Failure Boundaries | Avoids enterprise billing fees; models hard-stop `RESOURCE_EXHAUSTED` behavior. | Approved (ADR-10) |
