# GitStore // Terminal-Grade Open Software Marketplace

> **The zero-overhead software discovery and distribution depot powered directly by GitHub and Firebase.**

GitStore enables software creators, indie developers, and open-source maintainers to showcase and distribute finished digital products directly to users through verified GitHub Releases.

GitStore requires **zero binary hosting**, operates on **two platforms only** (GitHub & Firebase), and targets **$0.00 / month operational cost under published free quotas** of the Firebase Spark Plan without an attached billing account.

---

## 🚀 Key Architectural Pillars (Hardened in Phase 0.1.6)

1. **Strict Two-Platform Confinement:**
   - **GitHub:** GitHub Pages (Static Storefront), GitHub Actions (Privileged Batch Processing Engine), GitHub Issues (Ingestion Queue), GitHub Releases (Direct Binary Distribution).
   - **Firebase:** Firebase Authentication (GitHub OAuth, Magic Links & Step-Up Reauthentication), Cloud Firestore (Authoritative Database), Firebase Security Rules (Granular RBAC, Immutability & Step-Up Freshness), Firebase App Check (reCAPTCHA Enterprise Web Provider with `isTokenAutoRefreshEnabled: true`).
   - *Strictly Excluded:* AWS, Vercel, Supabase, Netlify, Cloud Functions, and custom backend servers.
2. **Framework-Free Vanilla Frontend:**
   - Pure HTML5, CSS3, and ES6 JavaScript Modules.
   - Zero bundlers, zero framework lock-in, 100% portable static web assets.
3. **Data Privacy, Public Profile Projection & Exhaustive Security Rules:**
   - User profile documents (`/users/{uid}`) are strictly private, accessible solely by the authenticated owner and batch worker, with client updates restricted via `diff().affectedKeys().hasOnly(...)`.
   - Public developer profiles are partitioned into an explicit public projection (`/public_profiles/{uid}`) with client-restricted, worker-managed reputation badges.
   - Internal collections (`/system_deployments`, `/system_config/deployment_state`, `/system_config/catalog_state`, `/admin_logs`) default to deny for all client access.
   - Abuse reports (`/reports`) require authenticated creators and enforce a 2000-character details limit with client read/update/delete blocked.
4. **Action-Bound Step-Up Authentication Proof Integrity (Option A):**
   - Privileged administrative commands require interactive fresh reauthentication via `reauthenticateWithPopup(auth.currentUser, new GithubAuthProvider())`.
   - Security Rules verify freshness at creation gate: `auth_time < 300s`, `sign_in_provider == 'github.com'`.
   - **Zero raw OAuth access tokens are stored in Firestore.** Requests attach an action-bound cryptographic proof binding caller UID, immutable numeric GitHub ID (`identities['github.com'][0]`), `auth_time`, action, target document, and action nonce.
   - Worker validates identity and canonical repo permissions (`Rohitpvt/GitStore`) within a 30-minute queue TTL from `createdAt`.
5. **Issue Ingestion Without Fake Accounts & Whitelisted Async Claiming:**
   - Issue Form submissions do not create fake Firebase Auth accounts; they record `submitterGithubId` and are claimed securely through an asynchronous `/claim_requests` queue.
   - Client creation of `/claim_requests` is strictly whitelisted via `hasOnly(['requestId', 'requesterUid', 'targetSubmissionId', 'status', 'createdAt'])`, blocking all privileged and worker-managed fields.
   - Submissions undergo strict ownership and release authority checks: Direct Personal Repository Owners qualify for auto-publication. Org members, committers, and community nominators are strictly held in `needs_review`.
6. **Globally Unique Deterministic Entity Hierarchy:**
   - Multiple distinct applications can originate from a single GitHub repository without collision, using distinct numeric product IDs (`prod_{repoNumericId}_{productSlug}`) and repo IDs (`gh_repo_{repoNumericId}`).
7. **12-State Recoverable Deployment State Machine & Recursive Canonical Digest:**
   - Governed by a formal 12-state state machine.
   - Manifest `data/deployment-manifest.json` contains `generatedAt`, `generationSequence`, and deterministic SHA-256 `catalogDigest` (omitting commit SHAs).
   - Digest algorithm uses recursive canonical JSON serialization (`canonicalizeJson`), preserving array order, rejecting non-finite numbers, and using null-byte length-prefixed framing (`sha256:v1:...`).
   - Full-history commit recovery on unshallow fetch (`git log origin/main`) with branch ancestry verification (`git merge-base --is-ancestor`) and paginated GitHub Commits API loop resolves post-commit failures.
8. **Official Pages API Status Lookup & Executable Case B Recovery:**
   - Evaluates deployment status using the official endpoint: `GET /repos/{owner}/{repo}/pages/deployments/{catalogCommitSha}` for status `"built"`.
   - In Case B (commit pushed to remote, but no Pages deployment exists), the worker executes an automated recovery path: checks out commit, validates digest, rebuilds artifact into `site/`, and dispatches Pages deployment.
9. **Durable Catalog Revisions & Transactional State Management:**
   - Monotonic revision tracking in `/system_config/catalog_state` updated inside an atomic `db.runTransaction` guarantees zero work loss for submissions arriving during active compilation or deployment runs.
10. **Least-Privilege Split-Job Pages Workflow, Cross-Job Handoff & Pre-Deploy Guard:**
    - Monolithic publish job split into `build-and-commit` (`contents: write`, `metadata: read`) and `deploy-to-pages` (`pages: write`, `id-token: write`, `contents: read`, `metadata: read` targeting `environment: github-pages`).
    - Explicit cross-job outputs handoff parameters across jobs.
    - Pre-deployment 4-case ordering guard asserts forward monotonic sequence progression immediately prior to `actions/deploy-pages@v4`.
    - Public assets staged strictly into an isolated `site/` directory (`actions/upload-pages-artifact@v3` packaging `path: site/`). Pinned 40-character commit SHA actions recommended.
11. **Static-First Read Architecture:**
    - Storefront visitors load pre-compiled `data/catalog.json` snapshots served directly by GitHub Pages.
    - Anonymous browsing generates **0 Firestore reads** and **0 App Check assessments**, keeping database usage under 3% of the free Firebase Spark quota.
12. **Binary Non-Execution & Provenance:**
    - GitStore never executes or sandboxes third-party binaries.
    - Download links point directly to cryptographic assets in authors' public GitHub Releases.

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
│       └── utils/                  # Sanitizers, formatters, semver helpers
├── data/
│   ├── deployment-manifest.json    # Deployed generation metadata & catalogDigest (no commit SHA)
│   ├── catalog.json                # Master pre-compiled product catalog
│   ├── categories.json             # Taxonomy definitions
│   └── products/                   # Granular product records (prod_{repoId}_{slug}.json)
├── scripts/
│   └── processor/
│       ├── index.js                # Main GitHub Actions batch entrypoint
│       ├── reconciler.js           # Full-history reconciler & Cases A-D recovery
│       ├── lease-manager.js        # Atomic transaction lease manager
│       ├── issue-parser.js         # Issue form ingestion & ownership proof
│       ├── moderation-handler.js   # Action-bound proof verification & canonical repo check
│       ├── github-verifier.js      # REST API repository & asset validator
│       ├── pre-deploy-guard.js     # Pre-deployment 4-case ordering assertion
│       └── catalog-compiler.js     # Static JSON generator & recursive canonical digest
├── docs/                           # Master Architectural Specifications (v1.7.0)
│   ├── ARCHITECTURE.md             # Core system design, interface contracts, ADR-01 through ADR-18
│   ├── DATABASE_SCHEMA.md          # Complete Firestore schema, reconciliation loop & security rules
│   ├── SECURITY.md                 # Threat model, RBAC, action-bound step-up proof, 18-row verification matrix
│   ├── SUBMISSION_PIPELINE.md      # Ingestion lifecycle, 12-state state machine, 11 failure scenarios
│   ├── AUTHENTICATION.md           # GitHub OAuth, magic links, App Check quotas, step-up proof
│   ├── DEPLOYMENT_GUIDE.md         # Operations guide, split-job workflow, secrets, pre-deploy guard
│   ├── TESTING_STRATEGY.md         # Testing pyramid, digest test vectors, emulator tests
│   ├── COST_AND_LIMITS.md          # Free-tier limits, hard-stop behavior, zero-cost analysis
│   ├── PRD.md                      # Product requirements document & personas
│   ├── SDD.md                      # Detailed system design document & sequence diagrams
│   ├── PHASE_PLAN.md               # Master phase tracking document (single source of execution truth)
│   ├── ROADMAP.md                  # Strategic implementation roadmap & Gantt schedule
│   ├── UI_DESIGN_SYSTEM.md         # Obsidian Dark Neo-Brutalist tokens & layout specs
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

All technical specifications are organized within the [`docs/`](./) directory:

| Document | Purpose | Version | Status |
|---|---|---|---|
| [`ARCHITECTURE.md`](./ARCHITECTURE.md) | High-level architecture, contracts, ADR-01 to ADR-18 | 1.7.0 | `IN_REVIEW` ⏳ |
| [`DATABASE_SCHEMA.md`](./DATABASE_SCHEMA.md) | Firestore collections, security rules, reconciler logic | 1.7.0 | `IN_REVIEW` ⏳ |
| [`SECURITY.md`](./SECURITY.md) | Threat model, action-bound step-up proof, 18-row verification matrix | 1.7.0 | `IN_REVIEW` ⏳ |
| [`SUBMISSION_PIPELINE.md`](./SUBMISSION_PIPELINE.md) | 12-state machine, ownership tiers, 11 failure scenarios | 1.7.0 | `IN_REVIEW` ⏳ |
| [`AUTHENTICATION.md`](./AUTHENTICATION.md) | OAuth, magic links, App Check quotas, step-up proof | 1.7.0 | `IN_REVIEW` ⏳ |
| [`DEPLOYMENT_GUIDE.md`](./DEPLOYMENT_GUIDE.md) | Split-job workflow, secrets, pre-deploy guard, setup guide | 1.7.0 | `IN_REVIEW` ⏳ |
| [`TESTING_STRATEGY.md`](./TESTING_STRATEGY.md) | Testing pyramid, digest test vectors, emulator tests | 1.7.0 | `IN_REVIEW` ⏳ |
| [`COST_AND_LIMITS.md`](./COST_AND_LIMITS.md) | Free-tier limits, hard-stop behavior, zero-cost analysis | 1.7.0 | `IN_REVIEW` ⏳ |
| [`PRD.md`](./PRD.md) | Product vision, target personas, functional requirements | 1.7.0 | `IN_REVIEW` ⏳ |
| [`SDD.md`](./SDD.md) | System design, sequence diagrams, component breakdowns | 1.7.0 | `IN_REVIEW` ⏳ |
| [`PHASE_PLAN.md`](./PHASE_PLAN.md) | Master development roadmap & phase control system | 1.6.0 | `IN_REVIEW` ⏳ |
| [`ROADMAP.md`](./ROADMAP.md) | Strategic Gantt chart & execution timeline | 1.7.0 | `IN_REVIEW` ⏳ |
| [`UI_DESIGN_SYSTEM.md`](./UI_DESIGN_SYSTEM.md) | Obsidian Dark Neo-Brutalist tokens & layout specs | 1.0.0 | `APPROVED` ✅ |

---

## 🚦 Project Status & Execution Governance

- **Current Active Phase:** **Phase 0.1.6 — Step-Up Proof Integrity, Firestore Rules & Deployment Reconciliation** (`IN_REVIEW` ⏳).
- **Execution Rule:** GitStore operates under strict phase-gate control. Implementation work on **Phase 1 (Frontend Core Foundation)** requires explicit user `APPROVED` sign-off.
- **Repository:** [`https://github.com/Rohitpvt/GitStore.git`](https://github.com/Rohitpvt/GitStore.git)
- **Primary Branch:** `main`
