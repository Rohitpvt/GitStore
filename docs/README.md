# GitStore // Terminal-Grade Open Software Marketplace

> **The zero-overhead software discovery and distribution depot powered directly by GitHub and Firebase.**

GitStore enables software creators, indie developers, and open-source maintainers to showcase and distribute finished digital products directly to users through verified GitHub Releases.

GitStore requires **zero binary hosting**, operates on **two platforms only** (GitHub & Firebase), and runs permanently at **$0.00 / month operational cost under published free quotas**.

---

## 🚀 Key Architectural Pillars (Hardened in Phase 0.1.3)

1. **Strict Two-Platform Confinement:**
   - **GitHub:** GitHub Pages (Static Storefront), GitHub Actions (Privileged Batch Processing Engine), GitHub Issues (Ingestion Queue), GitHub Releases (Direct Binary Distribution).
   - **Firebase:** Firebase Authentication (GitHub OAuth & Magic Links), Cloud Firestore (Authoritative Database), Firebase Security Rules (Granular RBAC & Immutability), Firebase App Check (reCAPTCHA Enterprise Web Provider with `isTokenAutoRefreshEnabled: true`).
   - *Strictly Excluded:* AWS, Vercel, Supabase, Netlify, Cloud Functions, and custom backend servers.
2. **Framework-Free Vanilla Frontend:**
   - Pure HTML5, CSS3, and ES6 JavaScript Modules.
   - Zero bundlers, zero framework lock-in, 100% portable static web assets.
3. **Decoupled Privileged Administration & Root Admin Bootstrap:**
   - Browser clients never receive Firebase Admin SDK credentials and cannot mint custom claims.
   - Privileged administrative commands write to an asynchronous `/moderation_requests` queue in Firestore.
   - Root administrator bootstrap matches canonical repository owner's immutable numeric GitHub account ID (`owner.id` from `GET /repos/Rohitpvt/GitStore`) against caller's linked `github.com` provider ID extracted via `admin.auth().verifyIdToken()`.
   - Maintainers are validated fail-closed against **GitStore's canonical repository** (`Rohitpvt/GitStore`) collaborator roles (`role_name === 'admin'` or `'maintain'`) before applying mutations via `firebase-admin`.
4. **Issue Ingestion Without Fake Accounts & Whitelisted Async Claiming:**
   - Issue Form submissions do not create fake Firebase Auth accounts; they record `submitterGithubId` and are claimed securely through an asynchronous `/claim_requests` queue.
   - Client creation of `/claim_requests` is strictly whitelisted via `hasOnly(['requestId', 'requesterUid', 'targetSubmissionId', 'status', 'createdAt'])`, blocking all privileged and worker-managed fields.
   - Submissions undergo strict ownership and release authority checks: Direct Personal Repository Owners qualify for auto-publication. Org members, committers, and community nominators are strictly held in `needs_review`.
   - Repository ownership does **not** automatically award the platform Verified Developer badge.
5. **Globally Unique Deterministic Entity Hierarchy:**
   - Multiple distinct applications can originate from a single GitHub repository without collision, using distinct numeric product IDs (`prod_{repoNumericId}_{productSlug}`) and repo IDs (`gh_repo_{repoNumericId}`).
6. **Non-Circular Manifest & Dedicated Pages Verification (Option A):**
   - Non-circular manifest: `data/deployment-manifest.json` contains `generationId` and monotonic `generationSequence` (omitting commit SHAs). The pair `(generationId, catalogCommitSha)` is persisted outside Git in Firestore `/system_deployments/{generationId}`.
   - Single-Workflow Execution (Option A): To overcome `GITHUB_TOKEN` push suppression, catalog compilation, Git commit/push, and Pages deployment (`actions/upload-pages-artifact@v3`, `actions/deploy-pages@v4`) run in the **same trusted workflow run** (`.github/workflows/batch-processor.yml`).
   - Dedicated Verification: Verifies status `"succeed"` via dedicated GitHub Pages endpoint `GET /repos/Rohitpvt/GitStore/pages/deployments/{id}` and confirms propagation via public HTTP manifest check.
   - Durable Failed Generation Retention: If deployment fails, the failure record is durably preserved in `/system_deployments/{generationId}` with attempt history, without prematurely resetting products to `approved`.
7. **Monotonic Generation Ordering & Durable Coalescing:**
   - Standard GitHub Actions queue discards intermediate pending runs; GitStore avoids runner queuing by coalescing all `approved` submissions into the next publication generation in Firestore.
   - Integer `generationSequence` prevents older generations from overwriting newer catalogs.
8. **Static-First Read Architecture:**
   - Storefront visitors load pre-compiled `data/catalog.json` snapshots served directly by GitHub Pages.
   - Anonymous browsing generates **0 Firestore reads** and **0 App Check assessments**, keeping database usage under 3% of the free Firebase Spark quota.
9. **Binary Non-Execution & Provenance:**
   - GitStore never executes or sandboxes third-party binaries.
   - Download links point directly to cryptographic assets in authors' public GitHub Releases.
10. **Workflow Event Allowlist & Fork Security:**
    - Privileged workflows execute exclusively on `schedule` and `workflow_dispatch` on the default branch.
    - `pull_request_target` and `workflow_run` are **strictly forbidden**, eliminating fork PR secret exfiltration and arbitrary code execution vulnerabilities.
11. **GitHub Pages Policy Pre-Launch Gate:**
    - Strictly respects GitHub’s prohibition against sending passwords on Pages by omitting traditional password forms.
    - Pre-launch written confirmation from GitHub Support is established as a formal launch gate.

---

## 📁 Repository Directory Structure

```text
GitStore/
├── .github/
│   ├── workflows/
│   │   └── batch-processor.yml     # Cron batch worker + Pages deployment (Option A)
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
│   ├── deployment-manifest.json    # Deployed generationId & generationSequence manifest (no SHA)
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

## 📚 Architectural Documentation Index (Phase 0.1.3 Hardened Baseline)

All 13 foundational architecture documents are synchronized and available in `docs/`:

| Document | Title | Purpose & Corrected Scope |
|---|---|---|
| [**PHASE_PLAN.md**](./PHASE_PLAN.md) | Master Phase Tracking Plan | Detailed lifecycle execution tracker, gate definitions, audit logs, and status. |
| [**PRD.md**](./PRD.md) | Product Requirements Document | User personas, multi-product formats, ownership framework, pre-launch gates. |
| [**SDD.md**](./SDD.md) | System Design Document | System components, sequence diagrams, recoverable two-tier publish protocol. |
| [**ARCHITECTURE.md**](./ARCHITECTURE.md) | Architecture Specification | System invariants, interface contracts, ADRs 01–12 (ADR-05, 07, 10 corrected; ADR-11, 12 added). |
| [**DATABASE_SCHEMA.md**](./DATABASE_SCHEMA.md) | Cloud Firestore Schema | `claim_requests` whitelisting, `/system_deployments`, non-circular manifest schema. |
| [**AUTHENTICATION.md**](./AUTHENTICATION.md) | Authentication & Identity | GitHub OAuth, Magic Links, reCAPTCHA Enterprise (`isTokenAutoRefreshEnabled: true`), admin bootstrap. |
| [**SUBMISSION_PIPELINE.md**](./SUBMISSION_PIPELINE.md) | Submission Pipeline | FSM states, Option A single-run deploy, 10 failure recovery scenarios, monotonic ordering. |
| [**SECURITY.md**](./SECURITY.md) | Security & Threat Model | Root admin bootstrap (`owner.id`), event security allowlist matrix, claim request whitelisting. |
| [**UI_DESIGN_SYSTEM.md**](./UI_DESIGN_SYSTEM.md) | UI Design System | Obsidian cyberpunk tokens, chamfered buttons, crosshairs, disclaimer panel. |
| [**ROADMAP.md**](./ROADMAP.md) | Implementation Roadmap | Phased execution milestones (Phase 0 to Phase 8) and exit criteria. |
| [**DEPLOYMENT_GUIDE.md**](./DEPLOYMENT_GUIDE.md) | Operations Manual | Setup for Pages, Actions secrets (`ADMIN_VERIFY_TOKEN`), single-workflow Option A setup. |
| [**TESTING_STRATEGY.md**](./TESTING_STRATEGY.md) | Testing Strategy | Collaborator mapping fixtures, collision tests, non-circular manifest & Pages `succeed` tests. |
| [**COST_AND_LIMITS.md**](./COST_AND_LIMITS.md) | Cost & Quota Analysis | Quota failure behaviors (`RESOURCE_EXHAUSTED`), reCAPTCHA Enterprise 10k allowance & billing conditions. |
| [**Sample Design Reference**](./Sample%20Design%20MD/brew-coffee-shop-DESIGN.md) | Design Token Specification | Reference cyberpunk hardware obsidian design tokens & components. |

---

## 🚦 Project Status: Phase 0.1.3 In Review

- **Phase 0 (Architecture Baseline & Documentation):** COMPLETE & APPROVED ✅
- **Phase 0.1 (Architecture Hardening & Critical Corrections):** `IN_REVIEW` ⏳
- **Phase 0.1.3 (Deployment Integrity, Admin Bootstrap & Workflow Security):** `IN_REVIEW` ⏳
- **Phase 1 (Frontend Core Foundation):** `NOT_STARTED` 🛑
- *Implementation Restriction:* In accordance with governance rules, **no application code has been implemented**. We await user approval of Phase 0.1 / 0.1.3 before proceeding to Phase 1.
