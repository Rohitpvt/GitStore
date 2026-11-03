# GitStore — Master Phase Plan & Development Control System

**Document Version:** 1.7.0  
**Last Updated:** 2026-11-03  
**Current Project State:** Phase 0.1.7 (Replay Protection, Firestore Field Integrity & Deployment State Consistency) — `IN_REVIEW` ⏳ | Awaiting User `APPROVED` Gate for Phase 1  
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
| **Phase 0.1.6**| Step-Up Proof Integrity, Firestore Rules & Deployment Reconciliation | `COMPLETED` ✅ | 100% (All 8 core objectives closed) | Hardened Baseline |
| **Phase 0.1.7**| Replay Protection, Firestore Field Integrity & Deployment State Consistency | `IN_REVIEW` ⏳ | 100% (All 10 core objectives closed) | User Review Pending |
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
- **Status:** `COMPLETED` ✅ (All 8 Core Objectives Closed; Hardened Baseline)
- **Objective:** Resolve remaining architectural weaknesses in step-up authentication proof integrity, exhaustive Firestore schema restrictions, deployment reconciliation query and execution paths, catalog digest generation, durable revision tracking, and pre-deployment ordering guards.

---

### Phase 0.1.7: Replay Protection, Firestore Field Integrity & Deployment State Consistency
- **Status:** `IN_REVIEW` ⏳ (All 10 Core Objectives Closed; Awaiting User Gate)
- **Objective:** Resolve remaining architectural and security defects identified during independent review of Phase 0.1.6:
  1. *GitHub Pages Deployment-Status Correction:* Replaced incorrect `status === 'built'` with documented `status === 'succeed'` on dedicated endpoint `GET /repos/{owner}/{repo}/pages/deployments/{pages_deployment_id}` (where parameter accepts numeric deployment ID or string commit SHA). Distinguish dedicated Pages API from generic deployments and builds. Schema updated to `pagesDeploymentReference: string | number | null`. Handle 200 (succeed, queued/in_progress, errored), 404 (Case B trigger), 401/403 (fatal permissions error), 5xx (transient). Unknown status never silently passes.
  2. *Repair Case B Recovery and Real Artifact Handoff:* In Case B (commit on remote, no Pages deployment), worker checks out commit, validates digest, rebuilds `site/`, returns `{ deployNeeded: true, ... }`, exports `deploy_needed=true` to `$GITHUB_OUTPUT`, Job 1 declares outputs, triggers `actions/upload-pages-artifact@v3`, and Job 2 (`deploy-to-pages`) runs `if: needs.build-and-commit.outputs.deploy_needed == 'true'`, runs pre-deploy guard, and deploys via `actions/deploy-pages@v4`.
  3. *Complete Firestore Submission Field Restrictions:* Removed worker-managed fields (`validationResults`, `ownershipProof`, `processingLease`, `adminNotes`, `reviewedBy`, `reviewedAt`, `publishedProductId`) from client creation allowlist in `/submissions/{submissionId}`. Client can only provide payload fields; trusted worker initializes worker fields via Firebase Admin SDK (which bypasses rules). Strict nested schema for `productData` and `selectedAsset`. Updates restricted via `diff().affectedKeys().hasOnly(['productData', 'status', 'updatedAt'])`.
  4. *Step-Up Nonce Replay Protection:* Introduced worker-managed collection `/consumed_stepup_nonces/{nonceHash}` (where `nonceHash = SHA256(actionNonce)`). Request claiming and nonce reservation execute in ONE atomic Firestore transaction. Second request with same nonce fails with `NONCE_ALREADY_CONSUMED`. Failed processing leaves nonce burned (recovery requires fresh reauthentication). Client access default-deny.
  5. *ActionHash Truthful Classification:* Classify client-generated SHA-256 `actionHash` as a **client data-integrity checksum**, NOT an independent cryptographic signature. True authorization rests on verified Firebase ID token, fresh `auth_time` (< 300s), GitHub numeric identity matching, and canonical collaborator check on `Rohitpvt/GitStore`.
  6. *Target Document Validation:* Worker derives canonical path from explicit action-to-collection mapping and validates `targetDocumentPath === expectedPath`, rejecting internal collections or mismatches.
  7. *Mixed-Provider Firebase Reauthentication:* Documented 9-step test scenario (Email sign-up -> link GitHub -> email sign-in -> `reauthenticateWithPopup(..., githubProvider)` -> inspect claims). Classify runtime provider-switching claim as `PENDING_INTEGRATION_TEST`. Require session authenticated via GitHub OAuth for admin actions.
  8. *Durable Catalog Revision Transaction & Stale Abort:* Require explicit `capturedRevision` (a non-negative number; NEVER substitute with `generationSequence`). Read `catalog_state` inside transaction. If stale (`capturedRevision < currentPublishedRevision`), **ABORT transaction immediately** without touching published sequence, records, or submission statuses. Mark record `status: 'stale_aborted'`.
  9. *Single Shared Generation Ordering Function:* Used one shared function `evaluateGenerationOrdering(artifactSeq, artifactGenId, activeSeq, activeGenId)` across generation creation, recovery, pre-deployment guard, and finalization.
  10. *Complete Commit Pagination & Error Classification:* Traverse `Link: rel="next"` without arbitrary 5-page limits. Distinguish empty search from incomplete search (rate limit/network -> `pending_retry`, never `abandoned`). Validate commit SHA format `/^[0-9a-f]{40}$/i` before shell commands.
- **Files Modified:** All 14 architecture, schema, security, and governance files across `docs/` and root repository.
- **Definition of Done (DoD):** All 10 problem areas formally closed, ADR-05, ADR-07, ADR-10, ADR-14 through ADR-18 refined, documentation synchronized, zero application code written, working tree verified clean.

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
| 2026-11-03 | 0.1.6 | Action-Bound Step-Up Proof Integrity (Zero Tokens) | Eliminated raw OAuth token storage in Firestore; cryptographically bound action proof `{ providerId, authTime, actionNonce, targetDocumentPath, actionHash }`; clarified 5-min creation gate vs 30-min queue TTL. | Approved (ADR-15) |
| 2026-11-03 | 0.1.6 | Exhaustive Security Rules Schema Whitelisting | Enforced `diff().affectedKeys().hasOnly()` on `/users` and `/public_profiles`; whitelisted `/submissions` and `/moderation_requests`; secured `/reports` with authenticated-only creation and 2000-char bounds. | Approved (ADR-14) |
| 2026-11-03 | 0.1.6 | Permanent Reconciliation Query & Case B Recovery | Fixed query to include all non-terminal reconciliation states; implemented real Case B recovery path; used official endpoint `GET .../pages/deployments/{catalogCommitSha}`; added Commits API pagination loop. | Approved (ADR-16) |
| 2026-11-03 | 0.1.6 | Recursive Canonical Catalog Digest & Vectors | Implemented recursive canonical JSON serializer (`canonicalizeJson`), null-byte length-prefixed framing, and defined test vectors for 6 core properties. | Approved (ADR-16) |
| 2026-11-03 | 0.1.6 | Durable Revision Transactions & Pre-Deploy Guard | Read and advance catalog revisions inside atomic `db.runTransaction`; implemented pre-deployment 4-case generation ordering guard before `deploy-pages`. | Approved (ADR-17, 18) |
| 2026-11-03 | 0.1.7 | GitHub Pages Dedicated Status Interpretation | Replaced `status === 'built'` with documented `status === 'succeed'` on endpoint `GET /repos/{owner}/{repo}/pages/deployments/{pages_deployment_id}` (accepts numeric deployment ID or string commit SHA). Distinguish dedicated Pages API from generic deployments and builds. Handled 200 (succeed, queued/in_progress, errored), 404 (Case B trigger), 401/403 (fatal permissions error), 5xx (transient). Schema updated to `pagesDeploymentReference: string \| number \| null`. Unknown status halts immediately. | In Review (ADR-16) |
| 2026-11-03 | 0.1.7 | Case B Recovery Real Cross-Job Artifact Handoff | Reconciler returns `{ deployNeeded: true, ... }` exported to `$GITHUB_OUTPUT`, Job 1 declares outputs, triggers `actions/upload-pages-artifact@v3`, Job 2 triggers conditionally (`deploy_needed == 'true'`) and deploys via `actions/deploy-pages@v4`. | In Review (ADR-16, 18) |
| 2026-11-03 | 0.1.7 | Submission Client Creation Whitelist Integrity | Stripped worker-managed fields (`validationResults`, `ownershipProof`, `processingLease`, `adminNotes`, `reviewedBy`, `reviewedAt`, `publishedProductId`) from client creation allowlist in `/submissions`. Client provides payload fields only; worker initializes via Admin SDK. | In Review (ADR-14) |
| 2026-11-03 | 0.1.7 | Step-Up Nonce Replay Protection Ledger | Introduced `/consumed_stepup_nonces/{nonceHash}` reserved atomically with moderation request claiming. Replays fail with `NONCE_ALREADY_CONSUMED`. Failed execution burns nonces. | In Review (ADR-15) |
| 2026-11-03 | 0.1.7 | ActionHash Truthful Classification | Classified `actionHash` as client data-integrity checksum (not cryptographic signature). True authorization verified via Firebase ID token, fresh `auth_time`, linked GitHub ID, and canonical repo collaborator check. | In Review (ADR-15) |
| 2026-11-03 | 0.1.7 | Canonical Target Document Path Validation | Worker derives expected canonical target document path from action type and asserts match, rejecting internal collections or ID mismatches. | In Review (ADR-15) |
| 2026-11-03 | 0.1.7 | Mixed-Provider Firebase Reauthentication Scenario | Documented 9-step verification scenario and classified runtime provider switching as `PENDING_INTEGRATION_TEST`. Mandated active GitHub session for admin actions. | In Review (ADR-15) |
| 2026-11-03 | 0.1.7 | Durable Revision Transaction with Stale Abort | Worker requires explicit non-negative `capturedRevision`. Reads `catalog_state` inside transaction and immediately aborts (`stale_aborted`) if `capturedRevision < currentPublishedRevision`. | In Review (ADR-17) |
| 2026-11-03 | 0.1.7 | Single Shared Generation Ordering Decision Function | Unified ordering logic across generation creation, recovery, pre-deployment guard, and finalization into `evaluateGenerationOrdering`. | In Review (ADR-16, 18) |
| 2026-11-03 | 0.1.7 | Complete Commit Pagination & Error Classification | Commits API loop traverses `Link: rel="next"` without arbitrary depth limits. Network/rate limits classified as `INCOMPLETE_SEARCH` (`pending_retry`), never `EMPTY_SEARCH`. SHA format validation prevents shell injection. | In Review (ADR-16) |
