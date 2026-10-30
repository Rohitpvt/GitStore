# GitStore // Terminal-Grade Open Software Marketplace

> **The zero-overhead software discovery and distribution depot powered directly by GitHub and Firebase.**

GitStore enables software creators, indie developers, and open-source maintainers to showcase and distribute finished digital products directly to users through verified GitHub Releases.

GitStore requires **zero binary hosting**, operates on **two platforms only** (GitHub & Firebase), and runs permanently at **$0.00 / month operational cost under published free quotas**.

---

## 🚀 Key Architectural Pillars (Hardened in Phase 0.1)

1. **Strict Two-Platform Confinement:**
   - **GitHub:** GitHub Pages (Static Storefront), GitHub Actions (Privileged Batch Processing Engine), GitHub Issues (Ingestion Queue), GitHub Releases (Direct Binary Distribution).
   - **Firebase:** Firebase Authentication (GitHub OAuth & Magic Links), Cloud Firestore (Authoritative Database), Firebase Security Rules (Granular RBAC & Immutability), Firebase App Check (reCAPTCHA v3 Web Provider).
   - *Strictly Excluded:* AWS, Vercel, Supabase, Netlify, Cloud Functions, and custom backend servers.
2. **Framework-Free Vanilla Frontend:**
   - Pure HTML5, CSS3, and ES6 JavaScript Modules.
   - Zero bundlers, zero framework lock-in, 100% portable static web assets.
3. **Decoupled Privileged Administration:**
   - Browser clients never receive Firebase Admin SDK credentials and cannot mint custom claims.
   - Privileged administrative commands write to an asynchronous `/moderation_requests` queue in Firestore.
   - The GitHub Actions batch worker authoritatively validates administrator identity via the GitHub REST API (`GET /repos/{owner}/{repo}/collaborators/{username}/permission`) before applying mutations via `firebase-admin`.
4. **Issue Ingestion & Ownership Verification:**
   - Submissions through GitHub Issue Forms undergo strict repository ownership checks (direct owner, public org member, or committer).
   - Submissions failing ownership proof are categorized as `community_nomination` and can never auto-publish.
5. **Two-Phase Publication Protocol:**
   - Guarantees database and static catalog consistency: Submissions transition to `approved`, static JSON catalogs are compiled and pushed, and only after GitHub Pages deployment confirms success does Firestore update status to `published`.
6. **Static-First Read Architecture:**
   - Storefront visitors load pre-compiled `data/catalog.json` snapshots served via GitHub Pages' global Fastly CDN.
   - Anonymous browsing generates **0 Firestore reads**, keeping database usage under 3% of the free Firebase Spark quota.
7. **Binary Non-Execution & Provenance:**
   - GitStore never executes or sandboxes third-party binaries.
   - Download links point directly to cryptographic assets in authors' public GitHub Releases.
8. **GitHub Pages Policy Pre-Launch Gate:**
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
│       └── product-submission.yml  # Git-native submission template
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
│   ├── catalog.json                # Pre-compiled static product index
│   ├── categories.json             # Taxonomy metadata and active counts
│   └── products/                   # Granular product detail JSON files
│       └── gh_{owner}_{repo}.json
├── docs/                           # Complete 13 Architectural Documents
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

## 📚 Architectural Documentation Index (Phase 0.1 Hardened Baseline)

All 13 foundational architecture documents are synchronized and available in `docs/`:

| Document | Title | Purpose & Hardened Scope |
|---|---|---|
| [**PRD.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/PRD.md) | Product Requirements Document | User personas, product formats, ownership framework, and pre-launch gates. |
| [**SDD.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/SDD.md) | System Design Document | System components, sequence diagrams, two-phase publication protocol. |
| [**ARCHITECTURE.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/ARCHITECTURE.md) | Architecture Specification | System invariants, interface contracts, and Architectural Decision Records (ADRs 01–10). |
| [**DATABASE_SCHEMA.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/DATABASE_SCHEMA.md) | Cloud Firestore Schema | `moderation_requests`, deterministic IDs (`gh_{owner}_{repo}`), lease pseudocode, security rules. |
| [**AUTHENTICATION.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/AUTHENTICATION.md) | Authentication & Identity | GitHub OAuth, Magic Links, reCAPTCHA v3, Pages policy analysis, account linking. |
| [**SUBMISSION_PIPELINE.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/SUBMISSION_PIPELINE.md) | Submission Pipeline | FSM states, batch worker engine, issue ownership verification, two-phase deploy. |
| [**SECURITY.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/SECURITY.md) | Security & Threat Model | Actor privilege matrix, decoupled admin security workflow, threat mitigations. |
| [**UI_DESIGN_SYSTEM.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/UI_DESIGN_SYSTEM.md) | UI Design System | Obsidian cyberpunk tokens, chamfered buttons, crosshairs, disclaimer panel. |
| [**ROADMAP.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/ROADMAP.md) | Implementation Roadmap | Phased execution milestones (Phase 0 to Phase 4) and exit criteria. |
| [**DEPLOYMENT_GUIDE.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/DEPLOYMENT_GUIDE.md) | Operations Manual | Step-by-step setup for GitHub Pages, Actions secrets, and Firebase Console. |
| [**TESTING_STRATEGY.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/TESTING_STRATEGY.md) | Testing Strategy | Local Firebase emulator rules testing, worker mocks, lease concurrency tests. |
| [**COST_AND_LIMITS.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/COST_AND_LIMITS.md) | Cost & Quota Analysis | Quota exhaustion behaviors (`RESOURCE_EXHAUSTED`), realistic free-tier math. |
| [**README.md**](file:///C:/Users/rghos/OneDrive%20-%20Vivekananda%20Institute%20of%20Professional%20Studies/PROJECTS/GitStore/docs/README.md) | Repository Readme | Project overview, navigation guide, and hardened architectural summary. |

---

## 🚦 Project Status: Phase 0.1 Audit Complete

- **Phase 0 (Architecture Baseline & Documentation):** COMPLETE ✅
- **Phase 0.1 (Architecture Hardening & Validation):** COMPLETE ✅
- **Phase 1 (Frontend Core Foundation):** PENDING APPROVAL ⏳
- *Implementation Restriction:* In accordance with Phase 0/0.1 directives, **no application code or production dependencies have been initialized**. Implementation begins upon Phase 1 authorization.
