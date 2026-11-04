# GitStore — Master Phase Plan & Development Control System

**Document Version:** 1.11.0  
**Last Updated:** 2026-11-04  
**Current Project State:** Phase 0.1.11 (Fan-Out Generation Fencing, Idempotent Retry Accounting & Schema Truthfulness) — `IN_REVIEW` 🔍 | Awaiting User `APPROVED` Gate for Phase 1  
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
| **Phase 0.1.7**| Replay Protection, Firestore Field Integrity & Deployment State Consistency | `COMPLETED` ✅ | 100% (All 10 core objectives closed) | Hardened Baseline |
| **Phase 0.1.8**| Node 24 Workflow Compatibility, Artifact Provenance & Transactional Deployment Integrity | `COMPLETED` ✅ | 100% (All 8 core audit items closed) | Hardened Baseline |
| **Phase 0.1.9**| Workflow Validation, Recovery SHA Preservation & Durable Finalization Recovery | `COMPLETED` ✅ | 100% (All 4 core defects + metadata verified) | Hardened Baseline |
| **Phase 0.1.10**| Fan-Out Ordering, Firestore Scalability & Recovery Error Integrity | `COMPLETED` ✅ | 100% (All 5 core problems resolved & verified) | Hardened Baseline |
| **Phase 0.1.11**| Fan-Out Generation Fencing, Idempotent Retry Accounting & Schema Truthfulness | `IN_REVIEW` ⏳ | 100% (All 8 core defect areas resolved & verified) | User Review Pending |
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
- **Status:** `COMPLETED` ✅ (All 10 Core Objectives Closed; Hardened Baseline)
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

### Phase 0.1.8: Node 24 Workflow Compatibility, Artifact Provenance & Transactional Deployment Integrity
- **Status:** `COMPLETED` ✅ (All 8 Core Audit Items Closed; Hardened Baseline)
- **Objective:** Resolve critical architectural, workflow runtime, staging provenance, output bridge, and finalization defects identified in the Phase 0.1.7 independent audit:
  1. *Update GitHub Actions to Supported Node 24 Runtimes:* GitHub removed Node 20 from hosted runners on September 23, 2026. Action metadata updated from deprecated Node 20 runtimes to official Node 24 releases, pinned to verified 40-character commit SHAs (`checkout@v5.0.0`, `setup-node@v7.1.0`, `upload-pages-artifact@v5.0.0`, `deploy-pages@v5.0.1`). Purged all stale `@v3`/`@v4` action tags.
  2. *Authoritative Case B Staging & Provenance:* Eliminated fragmented staging and corrupting workflow shell copy. Designed single authoritative staging procedure inside `processor/reconciler.js` via `git archive ${catalogCommitSha} ... | tar -x -C site/`, recomputing and asserting `stagedDigest === catalogDigest`.
  3. *Complete GitHub Actions Output Bridge:* Implemented robust output bridge function `emitStepOutput(name, value)` appending key-value pairs directly to `$GITHUB_OUTPUT`. Exported 6 parameters.
  4. *Two-Tier Finalization Architecture & Monotonic Guard:* Replaced bloated monolithic transaction with a 2-tier finalization architecture respecting Firestore's 500-write limit: Tier 1 3-write state anchor transaction + Tier 2 batched writes in chunks of 400.
  5. *Exhaustive Action-Specific Moderation Payload Validation:* Defined action-specific payload validation functions (`validModerationPayload(action, payload)`) across all 6 moderation actions.
  6. *ActionHash & Nonce Ledger Consistency:* Standardized on canonical `requestedByUid` across schema, rules, and processor code.
  7. *Commit Recovery Granular Error Classification:* Candidate manifest 404 classified as benign `CANDIDATE_NO_MANIFEST`. Network/rate limit errors set `isSearchIncomplete: true` and preserve `pending_retry`.
  8. *Pages API Status & Public Manifest HTTP Verification:* Dedicated endpoint `GET /repos/{owner}/{repo}/pages/deployments/{id}` checks `status === 'succeed'`. Firestore finalization deferred until public manifest HTTP check succeeds.
- **Files Modified:** All 14 architecture, schema, security, and governance files across `docs/` and root repository.
- **Definition of Done (DoD):** All 8 core audit items closed, ADR-15 through ADR-18 refined, documentation synchronized, zero application code written, working tree verified clean.

---

### Phase 0.1.9: Workflow Validation, Recovery SHA Preservation & Durable Finalization Recovery
- **Status:** `COMPLETED` ✅ (All 4 Core Defects + Metadata Verified; Hardened Baseline)
- **Objective:** Resolve critical workflow permissions, SHA preservation, fan-out recovery, and digest scope defects identified in the Phase 0.1.8 independent review:
  1. *Fix GitHub Actions Permissions:* Purged unsupported permission key `metadata: read` from both `build-and-commit` and `deploy-to-pages` jobs.
  2. *Preserve Case B Recovered Commit SHA:* Decoupled `commit_needed` from `deploy_needed`, preserving recovered commit SHA via `resolve-sha`.
  3. *Durable 2-Tier Submission Fan-Out Recovery:* Modeled fan-out with cursor resumption and 5-attempt fallback to `manual_review`.
  4. *Unified Canonical Catalog Digest Input Scope:* Standardized `computeCatalogDigest(catalogDataRoot)`.
  5. *Action Metadata Verification:* Accurately documented `upload-pages-artifact@v5.0.0` composite dependency on `upload-artifact@v7.0.1`.
- **Definition of Done (DoD):** All 4 defects resolved, ADR-16 through ADR-18 refined, 17-row verification matrix populated with truthful classifications, documentation synchronized to v1.9.0.

---

### Phase 0.1.10: Fan-Out Ordering, Firestore Scalability & Recovery Error Integrity
- **Status:** `COMPLETED` ✅ (All 5 Core Problems Resolved & Documented; Hardened Baseline)
- **Objective:** Resolve the 5 critical architectural and operational problems identified in the Phase 0.1.9 independent review:
  1. *Prevent Stale Submission Fan-Out:* Implement generation-aware, state-aware conditional fan-out in `executeSubmissionFanout()`. Use Firestore transactions enforcing: status eligibility (`status in ['approved', 'published']`; never resurrecting rejected, delisted, or withdrawn submissions), strictly monotonic generation sequence advancement (`genData.generationSequence > (currentSub.publishedGenerationSequence || 0)`), and idempotent replays across all 9 concurrency scenarios.
  2. *Remove Unbounded Submission ID Array:* Eliminate unbounded `expectedSubmissionIds: string[]` from the `/system_deployments/{generationId}` parent document, retaining strictly $O(1)$ scalar summary counters (`expectedSubmissionCount`, `completedSubmissionCount`, `submissionFanoutStatus`, `totalFanoutBatches`, `completedFanoutBatches`, `fanoutAttempts`). Store partitioned chunks in bounded subcollection `/system_deployments/{generationId}/fanout_batches/{batchId}` capped at 400 IDs each (~13 KB << 1 MiB limit).
  3. *Strict Commit Recovery Error Classification:* Reconciler error handler classifies candidate commit lookups explicitly: strictly treating confirmed HTTP 404 as benign missing manifests (`CANDIDATE_NO_MANIFEST`). All 401 auth failures, 403/429 rate limits, 5xx server errors, network timeouts, and connection resets set `isSearchIncomplete = true`. Malformed JSON is quarantined as `CANDIDATE_CORRUPT_MANIFEST` (`hasCorruptCandidates = true`). `EMPTY_SEARCH` can only be declared after a 100% full scan with zero interruptions and zero corrupt candidates.
  4. *Complete Nested `productData` Validation:* Exhaustively validate all 16 nested fields in Firestore Security Rules and batch worker: basic info (`name`, canonical regex `slug`, `repoUrl`, `repoOwner`, `repoName`, `releaseTag`, `shortDescription`, `fullDescription`, `category`, `version`, `license`), full `selectedAsset` map (all 5 keys, 500 MiB upper limit), media (HTTPS screenshot array <= 6), and platforms (`supportedPlatforms` enum). Reject arbitrary nested keys via `hasOnly()`.
  5. *Evaluate & Upgrade to `actions/checkout@v7.0.1`:* Pinned to official verified SHA `3d3c42e5aac5ba805825da76410c181273ba90b1`. Runs natively on Node 24 (`runs.using: 'node24'`, runner `>= v2.327.1`). Verified official documentation confirming native Git CLI operations (`git push origin main`) succeed with default `persist-credentials: true` via runner git credential helper, superseding outdated claims regarding `.git/config` breakage.
- **Files Modified:** All 11 architecture, schema, security, and governance files across `docs/` and root repository.
- **Definition of Done (DoD):** All 5 problems resolved, ADR-17 and ADR-18 updated, ADR-19 and ADR-20 added, 18-row verification matrix populated with truthful classifications, documentation synchronized to v1.10.0, zero application code written, working tree verified clean.

---

### Phase 0.1.11: Fan-Out Generation Fencing, Idempotent Retry Accounting & Schema Truthfulness
- **Status:** `IN_REVIEW` ⏳ (All 8 Core Defect Areas Resolved & Documented; Awaiting User Gate)
- **Objective:** Resolve the 8 critical architectural defects identified in the Phase 0.1.10 independent review:
  1. *Global Fan-Out Generation Fencing:* In `executeSubmissionFanout()`, read `/system_config/deployment_state` inside every mini-chunk transaction. If $genSeq < latestPublishedSequence$, abort fan-out as stale (`STALE_GLOBAL_GENERATION_ABORT`), strictly preventing older generations from modifying unreached submissions after a newer generation publishes. Fail closed on equal-sequence conflicting generation IDs (`HALT_CONFLICT`).
  2. *Post-Capture Moderation Revision Fence:* Add `approvedCatalogRevision` to `SubmissionDocument`. Mini-chunk transaction asserts `currentSub.approvedCatalogRevision <= genData.capturedRevision`, skipping submissions modified or re-approved after generation capture (`skipped_post_capture_reapproval`).
  3. *Product-Level Publication Eligibility Check:* Read `/products/{productId}` in mini-chunk transaction; skip fan-out activation if product status is `delisted` or `flagged`, ensuring product status is never overridden by submission metadata.
  4. *Pure Transaction Callbacks & Accurate Counter Accounting:* Purge external variable mutations and logging from `db.runTransaction` callbacks. Return outcome object `{ chunkCompleted, chunkSkipped }`, accumulating into caller counters only after commit succeeds.
  5. *Deterministic Batch Producer (`prepareFanoutBatches`):* Deduplicate expected submission IDs via Set, validate IDs, sort lexicographically, partition into chunks of max 400 IDs, write child documents with merge, read back to verify all batches exist and match counts, and set `fanoutPreparationStatus: 'prepared'`. Invariant: Tier 1 publication transaction must verify `fanoutPreparationStatus === 'prepared'` before advancing generation to `published`.
  6. *Atomic Batch Completion & Parent Counter Synchronization:* Child batch transition to `status: 'completed'` and parent summary increments (`completedFanoutBatches += 1`, `completedSubmissionCount += completed + skipped`) execute in a single atomic transaction. On 5 retry failures or fatal errors, atomically transition child batch to `status: 'manual_review'` and parent to `submissionFanoutStatus: 'manual_review'`, logging to `/admin_logs`.
  7. *Bounded Cursor Pagination & Concurrency Ownership:* Replace unpaginated `.get()` with cursor pagination (`.limit(10)`, `.orderBy('batchIndex', 'asc')`, `.startAfter(lastDoc)`). Worker ownership guaranteed by GitHub Actions root concurrency group `github-pages` plus atomic 15-minute CAS batch leases (`leaseOwner`, `leaseExpiresAt`).
  8. *Schema Truthfulness & Scalability Bounds:* Replace absolute "infinite horizontal scalability" claims with quota-aware bounds (max 128-char submission ID, 400 IDs $\approx 55\text{ KB} \ll 1\text{ MiB}$ limit). Explicitly distinguish Rules structural bounds from trusted worker authoritative checks (SemVer 2.0.0, SPDX License List, platform non-duplication via `arePlatformsUnique`, release asset validation via GitHub REST API). Synchronize all 11 project document headers to Version 1.11.0, Phase 0.1.11 `IN_REVIEW`.
- **Files Modified:** All 11 architecture, schema, security, and governance files across `docs/` and root repository.
- **Definition of Done (DoD):** All 8 defect areas resolved, ADR-17 and ADR-20 updated, ADR-21 and ADR-22 added, 19-row verification matrix populated with truthful classifications, documentation synchronized to v1.11.0, zero application code written, working tree verified clean.

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
| 2026-11-03 | 0.1.7 | Complete Commit Pagination & Error Classification | Commits API loop traverses `Link: rel="next"` without arbitrary depth limits. Network/rate limits classified as `INCOMPLETE_SEARCH` (`pending_retry`), never `EMPTY_SEARCH`. SHA format validation prevents shell injection. | Approved (ADR-16) |
| 2026-11-04 | 0.1.8 | Node 24 Actions Runner Migration | GitHub removed Node 20 from hosted runners on Sep 23, 2026. Action metadata migrated to Node 24 pinned by 40-char SHA (`checkout@v5.0.0`, `setup-node@v7.1.0`, `upload-pages-artifact@v5.0.0`, `deploy-pages@v5.0.1`). | In Review (ADR-18) |
| 2026-11-04 | 0.1.8 | Authoritative Case B Staging & Provenance | Eliminated corrupting workflow shell copy step (`cp -r data/* site/data/`). Reconciler is single authoritative staging mechanism: purges `site/`, extracts from target commit via `git archive ... | tar -x -C site/`, asserts `stagedDigest === catalogDigest`, and preserves `site/` intact. | In Review (ADR-16, 18) |
| 2026-11-04 | 0.1.8 | Robust Output Bridge (`emitStepOutput`) | Implemented step output bridge with delimiter escaping for multi-line protection, exporting 6 parameters to `$GITHUB_OUTPUT`. | In Review (ADR-18) |
| 2026-11-04 | 0.1.8 | Two-Tier Finalization Architecture | Replaced 500-write breaching transaction with Tier 1 (3-write state anchor transaction reading all state before writes) + Tier 2 (idempotent batched writes in chunks of 400). | In Review (ADR-17) |
| 2026-11-04 | 0.1.8 | Action-Specific Moderation Payload Schemas | Implemented `validModerationPayload(action, payload)` and helper schemas in Security Rules and processor logic for all 6 moderation actions. | In Review (ADR-14, 15) |
| 2026-11-04 | 0.1.8 | Canonical `requestedByUid` & Nonce Hash ID | Standardized on `requestedByUid` across schema, rules, and code. Document ID equals SHA-256 nonce hash. | In Review (ADR-15) |
| 2026-11-04 | 0.1.8 | Commit Recovery Granular Error Classification | Candidate 404 classified as benign `CANDIDATE_NO_MANIFEST`. Network/rate limit errors set `isSearchIncomplete: true` and preserve `pending_retry`. | In Review (ADR-16) |
| 2026-11-04 | 0.1.8 | Pages API Dedicated Status & HTTP Verification | Dedicated Pages API endpoint verifies `status === 'succeed'`. Firestore finalization deferred until public manifest HTTP check confirms CDN propagation. | Approved (ADR-16) |
| 2026-11-04 | 0.1.9 | Actions Permissions Correction | Purged unsupported `metadata: read` from workflow YAML. Configured least-privilege permissions: `contents: write` for `build-and-commit`; `pages: write`, `id-token: write`, `contents: read` for `deploy-to-pages`. | In Review (ADR-18) |
| 2026-11-04 | 0.1.9 | Case B Recovered SHA Preservation | Decoupled `commit_needed` from `deploy_needed`. Case B exports `commit_needed = false`, skipping the commit step and preserving recovered commit SHA unchanged via `resolve-sha`. | In Review (ADR-16, 18) |
| 2026-11-04 | 0.1.9 | Durable 2-Tier Submission Fan-Out Recovery | Modeled submission fan-out as independently recoverable operation with `submissionFanoutStatus`, cursor resumption, `expectedSubmissionIds`, and 5-attempt fallback to `manual_review`. | In Review (ADR-17) |
| 2026-11-04 | 0.1.9 | Unified Canonical Catalog Digest Input Scope | Standardized `computeCatalogDigest(catalogDataRoot)` over `catalog.json`, `categories.json`, `products/*.json`, strictly excluding manifest and static web assets. | Approved (ADR-16) |
| 2026-11-04 | 0.1.9 | Verified Action Release Metadata & Dependencies | Verified `upload-pages-artifact@v5.0.0` uses `actions/upload-artifact@v7.0.1` internally. Documented technical justification for retaining `checkout@v5.0.0` over `v6.x`. | Approved (ADR-18) |
| 2026-11-04 | 0.1.10 | Conditional Monotonic Submission Fan-Out | Enforced status eligibility (`approved` only; never resurrecting rejected/delisted submissions) and strictly monotonic sequence advance (`publishedGenerationSequence`) across all 9 concurrency scenarios. | In Review (ADR-17) |
| 2026-11-04 | 0.1.10 | Bounded Fan-Out Subcollection Storage | Removed unbounded `expectedSubmissionIds` array from parent document. Partitioned fan-out work into bounded subcollection `/system_deployments/{generationId}/fanout_batches/{batchId}` (max 400 IDs, ~13 KB << 1 MiB). Parent document holds $O(1)$ scalar summary counters. | In Review (ADR-17) |
| 2026-11-04 | 0.1.10 | Strict Commit Recovery Error Classifier | Only HTTP 404 is benign skip (`CANDIDATE_NO_MANIFEST`). 401, 403, 429, 5xx, timeouts, resets set `isSearchIncomplete = true`. Malformed JSON quarantined as `CANDIDATE_CORRUPT_MANIFEST`. `EMPTY_SEARCH` declared ONLY on 100% clean scans. | In Review (ADR-19) |
| 2026-11-04 | 0.1.10 | Exhaustive Nested `productData` Schema Validation | Implemented deep structural validation for all 16 fields in Firestore Rules and worker: basic info, canonical regex slug, full `selectedAsset` map (all 5 keys, 500 MiB limit), media, platform enums, and key allowlisting (`hasOnly`). | In Review (ADR-20) |
| 2026-11-04 | 0.1.10 | Upgrade to Actions Checkout v7.0.1 | Upgraded to official `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1` (Node 24, runner `>= v2.327.1`). Verified credential persistence documentation confirming native `git push` works seamlessly via runner credential helper. | In Review (ADR-18) |
