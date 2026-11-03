# GitStore — Master Phase Plan & Development Control System

**Document Version:** 1.6.0  
**Last Updated:** 2026-11-03  
**Current Project State:** Phase 0.1.6 (Step-Up Proof Integrity, Firestore Rules & Deployment Reconciliation) — `IN_REVIEW` ⏳ | Awaiting User `APPROVED` Gate for Phase 1  
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
| **Phase 0.1.4**| Evidence Alignment, Deployment Recovery & Governance Consistency | `COMPLETED` ✅ | 100% (All 8 core objectives closed) | Reconciled Baseline |
| **Phase 0.1.5**| Final Security Rules, Administrator Step-Up Authentication & Deployment Recovery Audit | `COMPLETED` ✅ | 100% (All 9 core objectives closed) | Hardened Baseline |
| **Phase 0.1.6**| Step-Up Proof Integrity, Firestore Rules & Deployment Reconciliation | `IN_REVIEW` ⏳ | 100% (All 8 core objectives closed) | User Review Pending |
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
- **Deliverables:** Complete foundational documentation files.

---

### Phase 0.1: Architecture Hardening & Final Validation
- **Status:** `IN_REVIEW` ⏳ (Audited, Hardened, and Validated)
- **Objective:** Audit Phase 0 architecture for structural, security, and operational gaps; correct trust boundaries; establish canonical repository collaborator checks; eliminate placeholder accounts; implement multi-product repository hierarchy; design self-healing publication loop; remove unapproved platform assumptions; correct reCAPTCHA quotas.

---

### Phase 0.1.4: Evidence Alignment, Deployment Recovery & Governance Consistency
- **Status:** `COMPLETED` ✅ (Superseded and Hardened by Phase 0.1.5 & Phase 0.1.6)
- **Objective:** Reconcile design claims with verified evidence, design deterministic deployment failure recovery, strengthen generation ordering, and synchronize governance documentation.

---

### Phase 0.1.5: Final Security Rules, Administrator Step-Up Authentication & Deployment Recovery Audit
- **Status:** `COMPLETED` ✅ (Superseded and Hardened by Phase 0.1.6)
- **Objective:** Execute exhaustive audit and hardening of Firestore Security Rules, administrator authentication, deployment recovery, concurrency, split-job workflows, pricing terms, and architecture decision records.

---

### Phase 0.1.6: Step-Up Proof Integrity, Firestore Rules & Deployment Reconciliation
- **Status:** `IN_REVIEW` ⏳ (All 8 Core Objectives Closed; Awaiting User Gate)
- **Objective:** Resolve remaining architectural weaknesses in step-up authentication proof integrity, exhaustive Firestore schema restrictions, deployment reconciliation query and execution paths, catalog digest generation, durable revision tracking, and pre-deployment ordering guards:
  1. *Action-Bound Step-Up Authentication Proof Integrity (Option A):* Completely eliminated raw `ephemeralGithubToken` storage in Firestore. Designed and specified action-bound proof structure `{ providerId: 'github.com', authTime, actionNonce, targetDocumentPath, actionHash }` where `actionHash = SHA256(action:target:nonce:authTime)`. Binding verified caller UID, immutable numeric GitHub ID (`identities['github.com'][0]`), `auth_time`, action, target document, and action nonce. Clarified the 5-minute creation gate in Security Rules (`auth_time < 300s`) versus the 30-minute queue execution TTL in the Actions worker.
  2. *Exhaustive Firestore Security Rules Schema Enforcement:*
     - `/users/{userId}`: create check, update protected via `diff().affectedKeys().hasOnly(['displayName', 'avatarUrl', 'bio', 'settings', 'updatedAt'])`, blocking modification of `uid`, `email`, `createdAt`, `isAdmin`, `isVerifiedDeveloper`, `reputationScore`, `githubNumericId`.
     - `/public_profiles/{userId}`: create check, update protected via `diff().affectedKeys().hasOnly(['displayName', 'avatarUrl', 'bio', 'updatedAt'])`, protecting worker-managed badges `isVerifiedDeveloper` and `publishedProductIds`.
     - `/submissions/{submissionId}`: exhaustive create whitelist, update protected via `diff().affectedKeys().hasOnly(['productData', 'status', 'updatedAt'])`, owner and status locked.
     - `/moderation_requests/{requestId}`: top-level and nested `stepUpProof` whitelist via `keys().hasOnly(...)`, complete client immutability (`allow update, delete: if false;`).
     - `/reports/{reportId}`: authenticated creation only (`request.auth != null`), strict schema whitelist, 2000-character details limit, client default-deny on read/update/delete.
     - Internal collections default-deny (`/system_deployments`, `/system_config/deployment_state`, `/system_config/catalog_state`, `/admin_logs`).
  3. *Permanent Deployment Reconciliation Fixes:*
     - *Query Bug Fix:* Expanded discovery query to include all non-terminal reconciliation states: `where('reconciliationStatus', 'in', ['none', 'sha_recovered', 'pages_dispatched', 'pending_retry'])`.
     - *Case B Recovery Implementation:* Replaced dummy `continue` with real executable recovery path: restores tree at `gen.catalogCommitSha`, validates digest against `gen.catalogDigest`, reconstructs `site/`, triggers artifact packaging and sets `deploy_needed = true`, updates `reconciliationStatus = 'pages_dispatched'`.
     - *Pages API Usage:* Removed undocumented list endpoint. Specified official endpoint `GET /repos/{owner}/{repo}/pages/deployments/{catalogCommitSha}` with documented error handling (200 built, 200 in_progress/queued, 200 errored, 404 Case B trigger, 401/403 permissions abort, 5xx retry).
     - *Full-History Discovery & Commits API Pagination:* Unshallow fetch, branch ancestry verification (`git merge-base --is-ancestor`), and a true pagination loop for GitHub Commits API fallback (`Link: rel="next"`).
  4. *Deterministic Recursive Canonical Catalog Digest:* Implemented `canonicalizeJson` with recursive Unicode key sorting, array order preservation, stable primitive formatting (rejecting `NaN`/`Infinity`), and length-prefixed null-byte framed boundaries (`FILE\0${relPath}\0${byteLength}\0`). Prefix `sha256:v1:...`. Provided test vectors for all 6 core properties.
  5. *Durable Catalog Revision Transactions:* Managed `/system_config/catalog_state` inside atomic `db.runTransaction`, reading current revisions inside transaction, enforcing monotonic advance, computing transactional `isDirty = (currentRequested > capturedRevision)`, and re-evaluating the 9 concurrency race conditions.
  6. *Pre-Deployment Generation Ordering Guard:* Enforced 4-case matrix evaluation immediately prior to `actions/deploy-pages@v4` in `deploy-to-pages` job.
  7. *Workflow & Cross-Job Handoff:* Explicitly declared cross-job outputs in `build-and-commit` consumed by `deploy-to-pages`. Recommended full 40-character commit SHA action pinning.
  8. *Governance & Documentation Synchronization:* Updated document headers across all 14 files to `IN_REVIEW — Awaiting Explicit User Approval`. Updated ADRs 14–18. Delivered exact 18-row verification matrix across 5 columns in `docs/SECURITY.md`.
- **Files Modified:** All 14 architecture, schema, security, and governance files across `docs/` and root repository.
- **Definition of Done (DoD):** All 8 problem areas formally closed, ADR-14 through ADR-18 refined, documentation synchronized, zero application code written, working tree verified clean.

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

---

### Phase 3: Developer Dashboard and Submission System
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Enable developers to submit software applications, inspect submission states, and manage their listings.

---

### Phase 4: GitHub Integration and Batch Processing
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Build the automated verification, leasing, and static JSON compilation engine using GitHub Actions.

---

### Phase 5: Admin Moderation and Public Catalog Publishing
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Implement the decoupled administrative moderation queue, developer verification, and Two-Phase catalog compilation.

---

### Phase 6: Security, Integration & Quality Assurance
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Perform end-to-end integration testing, full security auditing, stress testing, and address the pre-launch policy gate.

---

### Phase 7: Controlled Beta Launch
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Deploy production environment and onboard initial cohort of open-source developers under live monitoring.

---

### Phase 8: Post-Launch Improvements (Future Backlog)
- **Status:** `NOT_STARTED` 🛑
- **Objective:** Evaluate and implement post-MVP enhancements strictly adhering to dual-platform constraints.

---

## 4. Change Control & Deviation Log

| Date | Phase | Deviation / Change | Architectural Justification | Approval Authority |
|---|---|---|---|---|
| 2026-10-30 | 0.1 | Added `/moderation_requests` command queue | Eliminates browser-held Admin SDK keys; routes admin authorization through GitHub repository collaborator checks. | Approved (ADR-05) |
| 2026-10-30 | 0.1.1 | Canonical GitStore Repository Trust Boundary | Collaborator checks verify admin permissions against `Rohitpvt/GitStore`. Extracts verified GitHub numeric ID from Firebase Auth Admin SDK. | Approved (ADR-05) |
| 2026-10-30 | 0.1.1 | Removal of Fake Placeholder Firebase Accounts | Issue submissions record `submitterGithubId` and `claimStatus: 'unclaimed'`. Real claiming occurs upon OAuth login. Prevents auth DB pollution. | Approved (ADR-06) |
| 2026-10-30 | 0.1.1 | Strict Ownership Tiers (Direct Personal Owner Only) | Only personal repo owners qualify for auto-publication. Org members and committers forced to `needs_review` due to lack of release proof. | Approved (ADR-06) |
| 2026-10-30 | 0.1.1 | Self-Healing Reverse Failure Reconciliation | `reconcilePublishedState()` runs at start of worker cycle to heal unconfirmed Firestore documents. | Approved (ADR-07) |
| 2026-10-30 | 0.1 | Pre-Launch Policy Gate for Pages | Replaces premature policy clearance with formal confirmation requirement before public launch. | Approved (ADR-08) |
| 2026-10-30 | 0.1.1 | Multi-Product Repository Decoupling | Decouples repos (`gh_repo_{owner}_{repo}`) from products (`prod_{owner}_{slug}`). Enables monorepos to offer multiple distinct tools. | Approved (ADR-09) |
| 2026-10-30 | 0.1.1 | Removal of Fastly Assumption & App Check Lazy-Loading | Removes Fastly references; relies strictly on GitHub Pages. Lazy-loads App Check on auth pages only to stay under 10k/month quota. | Approved (ADR-10) |
| 2026-10-30 | 0.1.4 | Evidence Alignment & Verification Matrix | Formalized 6 evidence classifications; stripped unverified runtime "enforced" claims. | Approved (ADR-05-13) |
| 2026-11-03 | 0.1.5 | Public Profile Separation & Private Data Privacy | Locked `/users/{uid}` private; added `/public_profiles/{uid}` projection; enforced default-deny on internal collections. | Approved (ADR-14) |
| 2026-11-03 | 0.1.5 | Mandatory Fresh GitHub Step-Up Reauthentication | Enforced interactive `reauthenticateWithPopup`, `auth_time < 300s`, ephemeral token verification and immediate destruction. | Approved (ADR-15) |
| 2026-11-03 | 0.1.5 | 12-State Recovery State Machine & Deterministic Digest | Replaced 10-commit limit with full-history search, branch ancestry check, `catalogDigest` SHA-256, Cases A-D recovery. | Approved (ADR-16) |
| 2026-11-03 | 0.1.5 | Durable Catalog Revisions (`catalog_state`) | Monotonic revision tracking guaranteeing work preservation for submissions arriving during deployment. | Approved (ADR-17) |
| 2026-11-03 | 0.1.5 | Split-Job Workflow & Isolated `site/` Staging | Split `build-and-commit` and `deploy-to-pages` jobs; isolated public assets into `site/`. | Approved (ADR-18) |
| 2026-11-03 | 0.1.6 | Action-Bound Step-Up Proof Integrity (Zero Tokens) | Eliminated raw OAuth token storage in Firestore; cryptographically bound action proof `{ providerId, authTime, actionNonce, targetDocumentPath, actionHash }`; clarified 5-min creation gate vs 30-min queue TTL. | In Review (ADR-15) |
| 2026-11-03 | 0.1.6 | Exhaustive Security Rules Schema Whitelisting | Enforced `diff().affectedKeys().hasOnly()` on `/users` and `/public_profiles`; whitelisted `/submissions` and `/moderation_requests`; secured `/reports` with authenticated-only creation and 2000-char bounds. | In Review (ADR-14) |
| 2026-11-03 | 0.1.6 | Permanent Reconciliation Query & Case B Recovery | Fixed query to include all non-terminal reconciliation states; implemented real Case B recovery path; used official endpoint `GET .../pages/deployments/{catalogCommitSha}`; added Commits API pagination loop. | In Review (ADR-16) |
| 2026-11-03 | 0.1.6 | Recursive Canonical Catalog Digest & Vectors | Implemented recursive canonical JSON serializer (`canonicalizeJson`), null-byte length-prefixed framing, and defined test vectors for 6 core properties. | In Review (ADR-16) |
| 2026-11-03 | 0.1.6 | Durable Revision Transactions & Pre-Deploy Guard | Read and advance catalog revisions inside atomic `db.runTransaction`; implemented pre-deployment 4-case generation ordering guard before `deploy-pages`. | In Review (ADR-17, 18) |
