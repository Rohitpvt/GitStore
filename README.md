# GitStore // Terminal-Grade Open Software Marketplace

> **The zero-overhead software discovery and distribution depot powered directly by GitHub and Firebase.**

GitStore enables software creators, indie developers, and open-source maintainers to showcase and distribute finished digital products directly to users through verified GitHub Releases.

GitStore requires **zero binary hosting**, operates on **two platforms only** (GitHub & Firebase), and runs permanently at **$0.00 / month operational cost under published free quotas**.

---

## 🚀 Key Architectural Pillars (Hardened in Phase 0.1.2)

1. **Strict Two-Platform Confinement:**
   - **GitHub:** GitHub Pages (Static Storefront), GitHub Actions (Privileged Batch Processing Engine), GitHub Issues (Ingestion Queue), GitHub Releases (Direct Binary Distribution).
   - **Firebase:** Firebase Authentication (GitHub OAuth & Magic Links), Cloud Firestore (Authoritative Database), Firebase Security Rules (Granular RBAC & Immutability), Firebase App Check (reCAPTCHA Enterprise Web Provider).
   - *Strictly Excluded:* AWS, Vercel, Supabase, Netlify, Cloud Functions, and custom backend servers.
2. **Framework-Free Vanilla Frontend:**
   - Pure HTML5, CSS3, and ES6 JavaScript Modules.
   - Zero bundlers, zero framework lock-in, 100% portable static web assets.
3. **Decoupled Privileged Administration via Canonical Repository:**
   - Browser clients never receive Firebase Admin SDK credentials and cannot mint custom claims.
   - Privileged administrative commands write to an asynchronous `/moderation_requests` queue in Firestore.
   - The GitHub Actions batch worker extracts the caller's verified GitHub numeric ID via Firebase Admin SDK (`providerData.find(p => p.providerId === 'github.com')`), resolves their login via GitHub REST API, and validates permissions fail-closed against **GitStore's canonical repository** (`Rohitpvt/GitStore`) collaborator roles (`role_name === 'admin'` or `'maintain'`) before applying mutations via `firebase-admin`.
4. **Issue Ingestion Without Fake Accounts & Secure Async Claiming:**
   - Issue Form submissions do not create fake Firebase Auth accounts; they record `submitterGithubId` and are claimed securely through an asynchronous `/claim_requests` queue validated authoritatively by the batch worker.
   - Submissions undergo strict ownership and release authority checks: Direct Personal Repository Owners qualify for auto-publication. Org members, committers, and community nominators are strictly held in `needs_review`.
   - Repository ownership does **not** automatically award the platform Verified Developer badge.
5. **Globally Unique Deterministic Entity Hierarchy:**
   - Multiple distinct applications can originate from a single GitHub repository without collision, using distinct numeric product IDs (`prod_{repoNumericId}_{productSlug}`) and repo IDs (`gh_repo_{repoNumericId}`).
6. **Recoverable Two-Tier Publication State Machine:**
   - Guarantees eventual database and static catalog consistency. Submissions transition through `approved` -> `catalog_committed` -> `deploying` -> `deployment_verified` -> `published`.
   - Verification requires both the GitHub Deployments API (`state: "success"`) and public HTTP verification of `data/deployment-manifest.json` (`generationId` and `commitSha`).
   - The startup `reconcilePublishedState()` loop heals unconfirmed database documents if deployment succeeds but worker crashes before database update.
7. **Static-First Read Architecture:**
   - Storefront visitors load pre-compiled `data/catalog.json` snapshots served directly by GitHub Pages.
   - Anonymous browsing generates **0 Firestore reads**, keeping database usage under 3% of the free Firebase Spark quota.
8. **Binary Non-Execution & Provenance:**
   - GitStore never executes or sandboxes third-party binaries.
   - Download links point directly to cryptographic assets in authors' public GitHub Releases.
9. **GitHub Pages Policy Pre-Launch Gate:**
   - Strictly respects GitHub’s prohibition against sending passwords on Pages by omitting traditional password forms.
   - Pre-launch written confirmation from GitHub Support is established as a formal launch gate.

---

## 📁 Repository Directory Structure

```text
GitStore/
├── .github/
│   ├── workflows/
│   │   ├── batch-processor.yml     # Cron batch worker (every 10m)
│   │   └── pages-deploy.yml        # GitHub Pages deployment workflow
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
│   ├── deployment-manifest.json    # Deployed generationId & commitSha manifest
│   ├── catalog.json                # Pre-compiled static product index
│   ├── categories.json             # Taxonomy metadata and active counts
│   └── products/                   # Granular product detail JSON files
│       └── prod_{repoId}_{slug}.json
├── docs/                           # Complete 13 Hardened Architectural Documents
├── scripts/
│   └── processor/                  # Modular batch processing engine
├── index.html                      # Storefront Homepage
├── explore.html                    # Search & Multi-Faceted Filter Engine
├── product.html                    # Dynamic Product Detail Viewer
├── submit.html                     # Developer Submission Wizard
├── dashboard.html                  # Developer Management Portal
├── admin.html                      # Administrator Moderation Portal
└── login.html                      # Authentication View (OAuth + Magic Link)
```

---

## 📚 Architectural Documentation Index (Phase 0.1.2 Hardened Baseline)

All 13 foundational architecture documents are synchronized and available in `docs/`:

| Document | Title | Purpose & Corrected Scope |
|---|---|---|
| [**PHASE_PLAN.md**](docs/PHASE_PLAN.md) | Master Phase Tracking Plan | Detailed lifecycle execution tracker, gate definitions, audit logs, and status. |
| [**PRD.md**](docs/PRD.md) | Product Requirements Document | User personas, multi-product formats, ownership framework, pre-launch gates. |
| [**SDD.md**](docs/SDD.md) | System Design Document | System components, sequence diagrams, recoverable two-tier publish protocol. |
| [**ARCHITECTURE.md**](docs/ARCHITECTURE.md) | Architecture Specification | System invariants, interface contracts, ADRs 01–10 (ADR-05 to 10 corrected). |
| [**DATABASE_SCHEMA.md**](docs/DATABASE_SCHEMA.md) | Cloud Firestore Schema | `claim_requests`, `moderation_requests`, unique keys (`prod_{repoId}_{slug}`), reconciler. |
| [**AUTHENTICATION.md**](docs/AUTHENTICATION.md) | Authentication & Identity | GitHub OAuth, Magic Links, reCAPTCHA Enterprise, async claiming workflow. |
| [**SUBMISSION_PIPELINE.md**](docs/SUBMISSION_PIPELINE.md) | Submission Pipeline | FSM states, batch worker engine, 10 failure recovery scenarios, two-tier deploy check. |
| [**SECURITY.md**](docs/SECURITY.md) | Security & Threat Model | Actor privilege matrix, canonical repo admin flow, collaborator role checks. |
| [**UI_DESIGN_SYSTEM.md**](docs/UI_DESIGN_SYSTEM.md) | UI Design System | Obsidian cyberpunk tokens, chamfered buttons, crosshairs, disclaimer panel. |
| [**ROADMAP.md**](docs/ROADMAP.md) | Implementation Roadmap | Phased execution milestones (Phase 0 to Phase 8) and exit criteria. |
| [**DEPLOYMENT_GUIDE.md**](docs/DEPLOYMENT_GUIDE.md) | Operations Manual | Setup for Pages, Actions secrets (`ADMIN_VERIFY_TOKEN`), canonical collaborators. |
| [**TESTING_STRATEGY.md**](docs/TESTING_STRATEGY.md) | Testing Strategy | Collaborator mapping fixtures, collision tests, reconciler recovery tests. |
| [**COST_AND_LIMITS.md**](docs/COST_AND_LIMITS.md) | Cost & Quota Analysis | Quota failure behaviors (`RESOURCE_EXHAUSTED`), reCAPTCHA Enterprise 10k allowance. |
| [**Sample Design Reference**](docs/Sample%20Design%20MD/brew-coffee-shop-DESIGN.md) | Design Token Specification | Reference cyberpunk hardware obsidian design tokens & components. |

---

## 🚦 Project Status: Phase 0.1.2 In Review

- **Phase 0 (Architecture Baseline & Documentation):** COMPLETE & APPROVED ✅
- **Phase 0.1 (Architecture Hardening & Critical Corrections):** `IN_REVIEW` ⏳
- **Phase 0.1.2 (Final Security & Architecture Corrections):** `IN_REVIEW` ⏳
- **Phase 1 (Frontend Core Foundation):** `NOT_STARTED` 🛑
- *Implementation Restriction:* In accordance with governance rules, **no application code has been implemented**. We await user approval of Phase 0.1 / 0.1.2 before proceeding to Phase 1.
