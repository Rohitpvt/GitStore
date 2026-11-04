# GitStore // Terminal-Grade Open Software Marketplace

> **The zero-overhead software discovery and distribution depot powered directly by GitHub and Firebase.**

GitStore enables software creators, indie developers, and open-source maintainers to showcase and distribute finished digital products directly to users through verified GitHub Releases.

GitStore requires **zero binary hosting**, operates on **two platforms only** (GitHub & Firebase), and targets **$0.00 / month operational cost under published free quotas** of the Firebase Spark Plan without an attached billing account.

---

## 🚀 Key Architectural Pillars (Hardened in Phase 0.1.12)

1. **Strict Two-Platform Confinement:**
   - **GitHub:** GitHub Pages (Static Storefront), GitHub Actions (Privileged Batch Processing Engine), GitHub Issues (Ingestion Queue), GitHub Releases (Direct Binary Distribution).
   - **Firebase:** Firebase Authentication (GitHub OAuth, Magic Links & Step-Up Reauthentication), Cloud Firestore (Authoritative Database), Firebase Security Rules (Granular RBAC, Immutability & Step-Up Freshness), Firebase App Check (reCAPTCHA Enterprise Web Provider with `isTokenAutoRefreshEnabled: true`).
   - *Strictly Excluded:* AWS, Vercel, Supabase, Netlify, Cloud Functions, and custom backend servers.
2. **Framework-Free Vanilla Frontend:**
   - Pure HTML5, CSS3, and ES6 JavaScript Modules.
   - Zero bundlers, zero framework lock-in, 100% portable static web assets.
3. **Data Privacy, Public Profile Projection & Client Whitelist Integrity:**
   - User profile documents (`/users/{uid}`) are strictly private, accessible solely by the authenticated owner and batch worker, with client updates restricted via `diff().affectedKeys().hasOnly(...)`.
   - Public developer profiles are partitioned into an explicit public projection (`/public_profiles/{uid}`) with client-restricted, worker-managed reputation badges.
   - Submissions (`/submissions/{submissionId}`) enforce a strict client creation allowlist (`keys().hasOnly(['id', 'developerUid', 'submissionSource', 'repoNumericId', 'repoId', 'targetProductId', 'productData', 'status', 'claimStatus', 'createdAt', 'updatedAt'])`). Client creation is blocked if any worker-managed fields (`validationResults`, `ownershipProof`, `processingLease`, `adminNotes`, `reviewedBy`, `reviewedAt`, `publishedProductId`) are present.
   - Internal collections (`/system_deployments`, `/system_config/deployment_state`, `/system_config/catalog_state`, `/admin_logs`, `/consumed_stepup_nonces`) default to deny for all client access.
4. **Action-Bound Step-Up Reauthentication, Action-Specific Payloads & Nonce Ledger:**
   - Privileged administrative commands require interactive fresh reauthentication via `reauthenticateWithPopup(auth.currentUser, new GithubAuthProvider())`.
   - Security Rules verify freshness at creation gate: `auth_time < 300s`, `sign_in_provider == 'github.com'`.
   - **Zero raw OAuth access tokens are stored in Firestore.** Requests attach an action-bound proof binding caller UID, immutable numeric GitHub ID (`identities['github.com'][0]`), `auth_time`, action, target document, and action nonce.
   - Firestore Security Rules and batch worker enforce action-specific payload validation schemas (`validModerationPayload(action, payload)`) across all 6 moderation actions (`APPROVE_SUBMISSION`, `REJECT_SUBMISSION`, `VERIFY_DEVELOPER`, `REVOKE_DEVELOPER`, `DELIST_PRODUCT`, `FEATURE_PRODUCT`), validating target IDs, bounds, and path bindings while prohibiting arbitrary nested keys.
   - The worker atomically claims the request AND records the single-use nonce in `/consumed_stepup_nonces/{nonceHash}` in ONE transaction using canonical field `requestedByUid`. Replays fail with `NONCE_ALREADY_CONSUMED`. Failed execution permanently burns nonces.
   - The client-generated `actionHash` is classified as a client data-integrity checksum (not a cryptographic signature). True authorization rests on verified Firebase ID token claims, fresh `auth_time`, linked GitHub ID, and canonical repository collaborator check on `Rohitpvt/GitStore`.
5. **Issue Ingestion Without Fake Accounts & Whitelisted Async Claiming:**
   - Issue Form submissions do not create fake Firebase Auth accounts; they record `submitterGithubId` and are claimed securely through an asynchronous `/claim_requests` queue.
   - Client creation of `/claim_requests` is strictly whitelisted via `hasOnly(['requestId', 'requesterUid', 'targetSubmissionId', 'status', 'createdAt'])`, blocking all privileged and worker-managed fields.
   - Submissions undergo strict ownership and release authority checks: Direct Personal Repository Owners qualify for auto-publication. Org members, committers, and community nominators are strictly held in `needs_review`.
6. **Globally Unique Deterministic Entity Hierarchy:**
   - Multiple distinct applications can originate from a single GitHub repository without collision, using distinct numeric product IDs (`prod_{repoNumericId}_{productSlug}`) and repo IDs (`gh_repo_{repoNumericId}`).
7. **12-State Recoverable Deployment State Machine & Granular Commit Recovery:**
   - Governed by a formal 12-state state machine.
   - Manifest `data/deployment-manifest.json` contains `generatedAt`, `generationSequence`, and deterministic SHA-256 `catalogDigest` (omitting commit SHAs).
   - Digest algorithm uses recursive canonical JSON serialization (`canonicalizeJson`), preserving array order, rejecting non-finite numbers, and using null-byte length-prefixed framing (`sha256:v1:...`).
   - Unified catalog digest scope: `computeCatalogDigest(catalogDataRoot)` evaluates `catalog.json`, `categories.json`, and `products/*.json`, strictly excluding `deployment-manifest.json`, HTML, CSS, JS, and image assets.
   - Full-history commit recovery on unshallow fetch (`git log origin/main`) with branch ancestry verification (`git merge-base --is-ancestor`) and complete pagination of the GitHub Commits API following `Link: rel="next"`. Commit SHA format validation (`/^[0-9a-f]{40}$/i`) prevents command injection.
   - Granular commit recovery error classification: candidate manifest 404 is benign (`CANDIDATE_NO_MANIFEST`), while 401 auth failures, 403/429 rate limits, 5xx server errors, network timeouts, and connection resets set `isSearchIncomplete: true` and preserve generation state as `pending_retry`. Malformed manifest JSON is quarantined as `CANDIDATE_CORRUPT_MANIFEST` (`hasCorruptCandidates = true`), routing to `manual_review`. `EMPTY_SEARCH` is declared ONLY on 100% complete, zero-error, zero-corrupt scans.
8. **Node 24 Actions, Verified Permissions, Upgrade to Checkout v7.0.1 & Case B SHA Preservation:**
   - Workflows use official Node 24 runtime Actions pinned to 40-character SHAs (`actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1` [v7.0.1], `actions/setup-node@0a44ba7841725637a54e280ec667c276c6cc814e` [v7.1.0], `actions/upload-pages-artifact@56afc609e74202658d3ffba0e8f6dda462b719fa` [v5.0.0], `actions/deploy-pages@d6db90164ac5ed86f2b6aed7e0febac5b3c0c03e` [v5.0.1]), compatible with runner `>= v2.327.1`.
   - Native Git CLI credential persistence: Official documentation confirms `actions/checkout@v7.0.1` securely isolates credentials in `$RUNNER_TEMP` while configuring the Git credential helper so native Git operations (`git push origin main`) succeed automatically with default `persist-credentials: true` and `contents: write`.
   - Permissions strictly purged of unsupported keys (`metadata: read` removed; `contents: write` for build/commit; `pages: write`, `id-token: write`, `contents: read` for deploy).
   - Decision separation: `commit_needed` is separated from `deploy_needed`. Case B recovery skips the commit step entirely, preserving the recovered commit SHA unchanged via a dedicated `resolve-sha` step.
   - Evaluates deployment status using dedicated endpoint `GET /repos/{owner}/{repo}/pages/deployments/{pages_deployment_id}` checking for `status === "succeed"`. Unknown status halts execution immediately.
9. **Bounded Fan-Out Subcollections, Work-Plan Integrity, Lease Ownership & Atomic Completion:**
   - Finalization is separated into a 2-tier architecture respecting Firestore quotas and document bounds:
     - **Tier 1 (State Anchor Transaction):** Reads `/system_config/catalog_state`, `/system_config/deployment_state`, and `/system_deployments/{generationId}` BEFORE writes. Requires `fanoutPreparationStatus === 'prepared'`. Atomically commits exactly 3 state documents. Stale generations abort immediately (`stale_aborted`).
     - **Tier 2 (Bounded Subcollection Fan-Out):** Work is partitioned into bounded subcollection documents (`/system_deployments/{generationId}/fanout_batches/{batchId}`) containing at most 400 IDs each (~55 KB conservative size, ~5.37% of 1 MiB limit). Parent generation document retains strictly $O(1)$ scalar summary counters (`expectedSubmissionCount`, `completedSubmissionCount`, `totalFanoutBatches`, `completedFanoutBatches`).
     - **Work-Plan Preparation Integrity:** `prepareFanoutBatches` strictly validates all input IDs without silent dropping (`INVALID_SUBMISSION_ID_ABORT`), deduplicates, sorts lexicographically, computes canonical `workPlanDigest`, asserts immutability once prepared (`WORKPLAN_MUTATION_CONFLICT`), prunes stale batches, creates batch docs with `batchDigest`, and verifies element-by-element before marking `prepared`.
     - **Fail-Closed Product & Revision Checks:** Missing product documents fail closed (`skipped_product_not_found`). Delisted and flagged products fail closed and are never published or reactivated. Submissions require a valid positive integer `approvedCatalogRevision <= capturedRevision`.
     - **Transactional Distributed Lease Protocol:** Batches are claimed exclusively via `acquireBatchLease()` inside a transaction using a unique worker execution ID and a 15-minute lease expiry. Lease ownership and validity are asserted before processing each mini-chunk and during the atomic batch completion transaction; expired or superseded leases fail closed (`LEASE_LOST_ABORT`).
     - **Global Generation Fence:** Mini-chunk transactions read `/system_config/deployment_state` at the same consistency boundary. If $genSeq < latestPublishedSequence$, fan-out aborts as stale (`STALE_GLOBAL_GENERATION_ABORT`), strictly preventing older generations from modifying unreached records once a newer generation publishes. Fails closed on equal-sequence conflicting IDs (`HALT_CONFLICT`).
     - **Pure Transaction Callbacks & Atomic Completion:** Callbacks mutate zero outer variables; child batch completion and parent summary increments commit in a single atomic transaction. Exhausted retries (5 attempts) transition child batch and parent to `manual_review` together, logging to `/admin_logs`.
     - **Bounded Cursor Pagination & Concurrency Ownership:** Batches are queried via `.limit(10)`, `.orderBy('batchIndex', 'asc')`, and `.startAfter(lastBatchDoc)`. Reconciler discovers incomplete published generations via cursor pagination (`limit(10)`, `startAfter(lastGenDoc)`), excluding manual-review records.
10. **Single Shared Generation Ordering Function & Split-Job Workflow:**
    - Unified ordering logic across generation creation, Case B recovery, pre-deployment guard, and finalization into one shared function `evaluateGenerationOrdering(artifactSeq, artifactGenId, activeSeq, activeGenId)`.
    - Split-job workflow: `build-and-commit` (`contents: write`) and `deploy-to-pages` (`pages: write`, `id-token: write`, `contents: read` targeting `environment: github-pages`).
    - Public assets staged strictly into an isolated `site/` directory (`actions/upload-pages-artifact@v5.0.0` packaging `path: site/`). Pinned 40-character commit SHA actions used.
11. **Static-First Read Architecture:**
    - Storefront visitors load pre-compiled `data/catalog.json` snapshots served directly by GitHub Pages.
    - Anonymous browsing generates **0 Firestore reads** and **0 App Check assessments**, keeping database usage under 3% of the free Firebase Spark quota.
12. **Binary Non-Execution & Provenance:**
    - GitStore never executes or sandboxes third-party binaries.
    - Download links point directly to cryptographic assets in authors' public GitHub Releases.
13. **Schema Truthfulness & Authoritative Validation Separation:**
    - Firestore Security Rules enforce practical structural bounds (`version` regex, `license` length, `arePlatformsUnique` index equality for 1-6 platforms, and `repoUrl == 'https://github.com/' + repoOwner + '/' + repoName`).
    - Trusted batch worker executes authoritative validations: parsing full SemVer 2.0.0 specifications, matching licenses against official SPDX catalogs, checking platform set uniqueness, and verifying release assets directly against the GitHub Releases API.

---

## 📂 Repository Structure

```text
GitStore/
├── .github/
│   ├── workflows/
│   │   └── batch-processor.yml     # Least-privilege split-job worker (build-and-commit & deploy-to-pages)
│   └── ISSUE_TEMPLATE/
│       ├── product-submission.yml  # Structured issue form
│       ├── bug_report.yml          # Bug report form
│       └── feature_request.yml     # Feature request form
├── assets/
│   ├── css/
│   │   ├── variables.css           # Design tokens (obsidian dark, neo-brutalist neons)
│   │   ├── base.css                # Base typography, blueprint grid, utility resets
│   │   ├── components/             # Reusable UI components (buttons, cards, badges)
│   │   └── pages/                  # Page-specific styling
│   └── js/
│       ├── config/                 # Public Firebase client config & App Check init
│       ├── api/                    # Lightweight API modules (Auth, Firestore, Catalog)
│       ├── components/             # Reusable DOM component renderers
│       └── utils/                  # Sanitizers, formatters, semver validators
├── data/
│   ├── deployment-manifest.json    # Deployed generation metadata & catalogDigest (no commit SHA)
│   ├── catalog.json                # Master pre-compiled product catalog
│   ├── categories.json             # Taxonomy definitions
│   └── products/                   # Granular product records (prod_{repoId}_{slug}.json)
├── scripts/
│   └── processor/
│       ├── index.js                # Main GitHub Actions batch entrypoint
│       ├── reconciler.js           # Complete-history reconciler & Cases A-D recovery
│       ├── lease-manager.js        # Atomic transaction lease manager
│       ├── issue-parser.js         # Issue form ingestion & ownership proof
│       ├── moderation-handler.js   # Action-bound proof verification, nonce ledger & canonical repo check
│       ├── github-verifier.js      # REST API repository & asset validator
│       ├── pre-deploy-guard.js     # Pre-deployment shared ordering assertion
│       └── catalog-compiler.js     # Static JSON generator & recursive canonical digest
├── docs/                           # Master Architectural Specifications (v1.11.0)
│   ├── ARCHITECTURE.md             # Core system design, interface contracts, ADR-01 through ADR-22
│   ├── DATABASE_SCHEMA.md          # Complete Firestore schema, bounded fanout batches, reconciler logic & security rules
│   ├── SECURITY.md                 # Threat model, RBAC, action-bound step-up auth, 19-row verification matrix
│   ├── SUBMISSION_PIPELINE.md      # Ingestion lifecycle, 12-state state machine, 12 failure scenarios
│   ├── AUTHENTICATION.md           # GitHub OAuth, magic links, App Check quotas, step-up proof
│   ├── DEPLOYMENT_GUIDE.md         # Operations guide, split-job workflow, secrets, pre-deploy guard
│   ├── TESTING_STRATEGY.md         # Testing pyramid, digest test vectors, rules unit tests, mocks
│   ├── COST_AND_LIMITS.md          # Spark quota math, zero-cost architecture analysis
│   ├── PRD.md                      # Product requirements document & personas
│   ├── SDD.md                      # Detailed system design document & sequence diagrams
│   ├── PHASE_PLAN.md               # Master phase tracking document (single source of execution truth)
│   ├── ROADMAP.md                  # Strategic implementation roadmap & Gantt schedule
│   ├── UI_DESIGN_SYSTEM.md         # Obsidian Neo-Brutalist design specifications
│   └── README.md                   # Documentation index & architectural summary
├── index.html                      # Homepage & featured showcase
├── explore.html                    # Search, category, and platform filtering
├── product.html                    # Dynamic product detail & download page
├── submit.html                     # Multi-format product submission wizard
├── dashboard.html                  # Developer submission management portal
├── admin.html                      # Moderation portal with action-bound step-up reauthentication
└── login.html                      # Authentication gateway (GitHub OAuth & Magic Link)
```

---

## 📚 Master Architectural Documentation Index

All technical specifications are organized within the [`docs/`](./docs) directory:

| Document | Purpose | Version | Status |
|---|---|---|---|
| [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md) | High-level architecture, contracts, ADR-01 to ADR-24 | 1.12.0 | `IN_REVIEW` ⏳ |
| [`docs/DATABASE_SCHEMA.md`](./docs/DATABASE_SCHEMA.md) | Firestore collections, security rules, bounded fanout batches, reconciler logic | 1.12.0 | `IN_REVIEW` ⏳ |
| [`docs/SECURITY.md`](./docs/SECURITY.md) | Threat model, action-bound step-up proof, 20-scenario verification matrix | 1.12.0 | `IN_REVIEW` ⏳ |
| [`docs/SUBMISSION_PIPELINE.md`](./docs/SUBMISSION_PIPELINE.md) | 12-state machine, ownership tiers, 12 failure scenarios | 1.12.0 | `IN_REVIEW` ⏳ |
| [`docs/AUTHENTICATION.md`](./docs/AUTHENTICATION.md) | OAuth, magic links, App Check quotas, step-up proof | 1.9.0 | `IN_REVIEW` ⏳ |
| [`docs/DEPLOYMENT_GUIDE.md`](./docs/DEPLOYMENT_GUIDE.md) | Node 24 workflow, secrets, pre-deploy guard, manifest v1.12.0 | 1.12.0 | `IN_REVIEW` ⏳ |
| [`docs/TESTING_STRATEGY.md`](./docs/TESTING_STRATEGY.md) | Testing pyramid, digest test vectors, emulator tests | 1.12.0 | `IN_REVIEW` ⏳ |
| [`docs/COST_AND_LIMITS.md`](./docs/COST_AND_LIMITS.md) | Free-tier limits, hard-stop behavior, zero-cost analysis | 1.9.0 | `IN_REVIEW` ⏳ |
| [`docs/PRD.md`](./docs/PRD.md) | Product vision, target personas, functional requirements | 1.9.0 | `IN_REVIEW` ⏳ |
| [`docs/SDD.md`](./docs/SDD.md) | System design, sequence diagrams, component breakdowns | 1.12.0 | `IN_REVIEW` ⏳ |
| [`docs/PHASE_PLAN.md`](./docs/PHASE_PLAN.md) | Master development roadmap & phase control system | 1.12.0 | `IN_REVIEW` ⏳ |
| [`docs/ROADMAP.md`](./docs/ROADMAP.md) | Strategic Gantt chart & execution timeline | 1.12.0 | `IN_REVIEW` ⏳ |
| [`docs/UI_DESIGN_SYSTEM.md`](./docs/UI_DESIGN_SYSTEM.md) | Obsidian Dark Neo-Brutalist tokens & layout specs | 1.1.0 | `APPROVED` ✅ |

---

## 🚦 Project Status & Execution Governance

- **Current Active Phase:** **Phase 0.1.12 — Fan-Out Work-Plan Integrity, Lease Ownership & Fail-Closed Recovery** (`IN_REVIEW` ⏳).
- **Execution Rule:** GitStore operates under strict phase-gate control. Implementation work on **Phase 1 (Frontend Core Foundation)** requires explicit user `APPROVED` sign-off.
- **Repository:** [`https://github.com/Rohitpvt/GitStore.git`](https://github.com/Rohitpvt/GitStore.git)
- **Primary Branch:** `main`
