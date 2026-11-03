# GitStore // Terminal-Grade Open Software Marketplace

> **The zero-overhead software discovery and distribution depot powered directly by GitHub and Firebase.**

GitStore enables software creators, indie developers, and open-source maintainers to showcase and distribute finished digital products directly to users through verified GitHub Releases.

GitStore requires **zero binary hosting**, operates on **two platforms only** (GitHub & Firebase), and targets **$0.00 / month operational cost under published free quotas** of the Firebase Spark Plan without an attached billing account.

---

## 🚀 Key Architectural Pillars (Hardened in Phase 0.1.5)

1. **Strict Two-Platform Confinement:**
   - **GitHub:** GitHub Pages (Static Storefront), GitHub Actions (Privileged Batch Processing Engine), GitHub Issues (Ingestion Queue), GitHub Releases (Direct Binary Distribution).
   - **Firebase:** Firebase Authentication (GitHub OAuth, Magic Links & Step-Up Reauthentication), Cloud Firestore (Authoritative Database), Firebase Security Rules (Granular RBAC, Immutability & Step-Up Freshness), Firebase App Check (reCAPTCHA Enterprise Web Provider with `isTokenAutoRefreshEnabled: true`).
   - *Strictly Excluded:* AWS, Vercel, Supabase, Netlify, Cloud Functions, and custom backend servers.
2. **Framework-Free Vanilla Frontend:**
   - Pure HTML5, CSS3, and ES6 JavaScript Modules.
   - Zero bundlers, zero framework lock-in, 100% portable static web assets.
3. **Data Privacy & Public Profile Projection:**
   - User profile documents (`/users/{uid}`) are strictly private, accessible solely by the authenticated owner and batch worker.
   - Public developer profiles are partitioned into an explicit public projection (`/public_profiles/{uid}`) with client-restricted, worker-managed reputation badges.
   - Internal collections (`/system_deployments`, `/system_config/deployment_state`, `/system_config/catalog_state`, `/admin_logs`) default to deny for all client access.
4. **Mandatory Fresh GitHub Step-Up Reauthentication (Option A):**
   - Privileged administrative commands require interactive fresh reauthentication via `reauthenticateWithPopup(auth.currentUser, new GithubAuthProvider())`.
   - Security Rules verify freshness: `auth_time < 300s`.
   - Ephemeral single-use, action-bound step-up proofs are verified via GitHub REST API `GET /user`, verified against canonical repo collaborator permissions (`Rohitpvt/GitStore`), and **immediately purged** (`FieldValue.delete()`), guaranteeing zero credential retention and strict anti-replay protection.
5. **Issue Ingestion Without Fake Accounts & Whitelisted Async Claiming:**
   - Issue Form submissions do not create fake Firebase Auth accounts; they record `submitterGithubId` and are claimed securely through an asynchronous `/claim_requests` queue.
   - Client creation of `/claim_requests` is strictly whitelisted via `hasOnly(['requestId', 'requesterUid', 'targetSubmissionId', 'status', 'createdAt'])`, blocking all privileged and worker-managed fields.
   - Submissions undergo strict ownership and release authority checks: Direct Personal Repository Owners qualify for auto-publication. Org members, committers, and community nominators are strictly held in `needs_review`.
6. **Globally Unique Deterministic Entity Hierarchy:**
   - Multiple distinct applications can originate from a single GitHub repository without collision, using distinct numeric product IDs (`prod_{repoNumericId}_{productSlug}`) and repo IDs (`gh_repo_{repoNumericId}`).
7. **12-State Recoverable Deployment State Machine & Deterministic Digest:**
   - Governed by a formal 12-state state machine.
   - Manifest `data/deployment-manifest.json` contains `generatedAt`, `generationSequence`, and deterministic SHA-256 `catalogDigest` (omitting commit SHAs).
   - Full-history commit recovery on unshallow fetch (`git log origin/main`) with branch ancestry verification (`git merge-base --is-ancestor`) and paginated GitHub Commits API fallback resolves post-commit failures.
   - Missing Pages deployment IDs are deterministically resolved across Cases A, B, C, and D.
8. **Durable Catalog Revisions (`catalog_state`):**
   - Durable monotonic revision tracking in `/system_config/catalog_state` guarantees zero work loss for submissions arriving during active compilation or deployment runs.
9. **Least-Privilege Split-Job Pages Workflow & Isolated Staging:**
   - Monolithic publish job split into `build-and-commit` (`contents: write`, `metadata: read`) and `deploy-to-pages` (`pages: write`, `id-token: write`, `contents: read`, `metadata: read` targeting `environment: github-pages`).
   - Public assets staged strictly into an isolated `site/` directory (`actions/upload-pages-artifact@v5` packaging `path: site/`).
   - Verified official 2026 releases: `actions/checkout@v7`, `actions/setup-node@v7`, `actions/upload-pages-artifact@v5`, `actions/deploy-pages@v5`.
10. **Static-First Read Architecture:**
    - Storefront visitors load pre-compiled `data/catalog.json` snapshots served directly by GitHub Pages.
    - Anonymous browsing generates **0 Firestore reads** and **0 App Check assessments**, keeping database usage under 3% of the free Firebase Spark quota.
11. **Binary Non-Execution & Provenance:**
    - GitStore never executes or sandboxes third-party binaries.
    - Download links point directly to cryptographic assets in authors' public GitHub Releases.
12. **Workflow Event Allowlist & Fork Security:**
    - Privileged workflows execute exclusively on `schedule` and `workflow_dispatch` on the default branch.
    - `pull_request_target` and `workflow_run` are **strictly forbidden**, eliminating fork PR secret exfiltration and arbitrary code execution vulnerabilities.
13. **GitHub Pages Policy Pre-Launch Gate:**
    - Strictly respects GitHub’s prohibition against sending passwords on Pages by omitting traditional password forms.
    - Pre-launch written confirmation from GitHub Support is established as a formal launch gate.

---

## 📁 Repository Directory Structure

```text
GitStore/
├── .github/
│   ├── workflows/
│   │   └── batch-processor.yml     # Least-privilege split-job worker (build-and-commit & deploy-to-pages)
│   └── ISSUE_TEMPLATE/
│       ├── product-submission.yml  # Git-native submission template
│       ├── bug_report.yml          # Bug report form
│       └── feature_request.yml     # Feature request form
├── assets/
│   ├── css/
│   │   ├── variables.css           # Obsidian cyberpunk design tokens
│   │   ├── base.css                # Global resets & typography
│   │   ├── components/             # Reusable chamfered buttons, cards, badges
│   │   └── pages/                  # Page-specific layout styles
│   └── js/
│       ├── config/                 # Public Firebase config & App Check init
│       ├── api/                    # Auth, Firestore, and Catalog fetch abstractions
│       ├── components/             # Navbar, product cards, filter bars
│       └── utils/                  # Sanitizers, formatters, semver helpers
├── data/
│   ├── deployment-manifest.json    # Deployed generationId, generationSequence, generatedAt & catalogDigest (no SHA)
│   ├── catalog.json                # Pre-compiled static product index
│   ├── categories.json             # Taxonomy metadata and active counts
│   └── products/                   # Granular product detail JSON files
│       └── prod_{repoId}_{slug}.json
├── docs/                           # Complete Hardened Architectural Documents
├── scripts/
│   └── processor/                  # Modular batch processing engine
├── index.html                      # Storefront Homepage
├── explore.html                    # Search & Multi-Faceted Filter Engine
├── product.html                    # Dynamic Product Detail Viewer
├── submit.html                     # Developer Submission Wizard
├── dashboard.html                  # Developer Management Portal
├── admin.html                      # Administrator Moderation Portal with Step-Up Reauth
└── login.html                      # Authentication View (OAuth + Magic Link)
```

---

## 📚 Architectural Documentation Index (Phase 0.1.5 Hardened Baseline)

All 13 foundational architecture documents are synchronized and available in `docs/`:

| Document | Title | Purpose & Corrected Scope |
|---|---|---|
| [**PHASE_PLAN.md**](docs/PHASE_PLAN.md) | Master Phase Tracking Plan | Detailed lifecycle execution tracker, gate definitions, audit logs, and status. |
| [**PRD.md**](docs/PRD.md) | Product Requirements Document | User personas, multi-product formats, ownership framework, pre-launch gates. |
| [**SDD.md**](docs/SDD.md) | System Design Document | System components, sequence diagrams, 12-state recoverable publish protocol. |
| [**ARCHITECTURE.md**](docs/ARCHITECTURE.md) | Architecture Specification | System invariants, interface contracts, ADRs 01–18 (ADRs 14-18 added; ADRs 05, 07, 10-13 refined). |
| [**DATABASE_SCHEMA.md**](docs/DATABASE_SCHEMA.md) | Cloud Firestore Schema | Private `/users`, public `/public_profiles`, `/system_config/catalog_state`, 12-state records, rules. |
| [**AUTHENTICATION.md**](docs/AUTHENTICATION.md) | Authentication & Identity | Option A Fresh Step-Up Reauthentication, token destruction, App Check reCAPTCHA Enterprise. |
| [**SUBMISSION_PIPELINE.md**](docs/SUBMISSION_PIPELINE.md) | Submission Pipeline | 12-state FSM, full-history recovery, deterministic digest, Cases A-D recovery, split-job workflow. |
| [**SECURITY.md**](docs/SECURITY.md) | Security & Threat Model | Step-Up challenge protocol, token destruction, 20-row comprehensive verification matrix. |
| [**UI_DESIGN_SYSTEM.md**](docs/UI_DESIGN_SYSTEM.md) | UI Design System | Obsidian cyberpunk tokens, chamfered buttons, crosshairs, disclaimer panel. |
| [**ROADMAP.md**](docs/ROADMAP.md) | Implementation Roadmap | Phased execution milestones (Phase 0 to Phase 8) and exit criteria. |
| [**DEPLOYMENT_GUIDE.md**](docs/DEPLOYMENT_GUIDE.md) | Operations Manual | Least-privilege split-job workflow setup, `@v7`/`@v5` action versions, `site/` staging directory. |
| [**TESTING_STRATEGY.md**](docs/TESTING_STRATEGY.md) | Testing Strategy | Private user doc lockout, public profile separation, step-up verification, digest matching. |
| [**COST_AND_LIMITS.md**](docs/COST_AND_LIMITS.md) | Cost & Quota Analysis | Free quotas under Spark plan without linked billing, failure behaviors (`RESOURCE_EXHAUSTED`). |
| [**Sample Design Reference**](docs/Sample%20Design%20MD/brew-coffee-shop-DESIGN.md) | Design Token Specification | Reference cyberpunk hardware obsidian design tokens & components. |

---

## 🚦 Project Status: Phase 0.1.5 In Review

- **Phase 0 (Architecture Baseline & Documentation):** COMPLETE & APPROVED ✅
- **Phase 0.1 (Architecture Hardening & Critical Corrections):** `IN_REVIEW` ⏳
- **Phase 0.1.3 (Deployment Integrity, Admin Bootstrap & Workflow Security):** COMPLETED ✅
- **Phase 0.1.4 (Evidence Alignment, Deployment Recovery & Governance Consistency):** COMPLETED ✅
- **Phase 0.1.5 (Final Security Rules, Step-Up Auth & Recovery Audit):** `IN_REVIEW` ⏳
- **Phase 1 (Frontend Core Foundation):** `NOT_STARTED` 🛑
- *Implementation Restriction:* In accordance with governance rules, **no application code has been implemented**. We await user approval of Phase 0.1 / 0.1.5 before proceeding to Phase 1.
