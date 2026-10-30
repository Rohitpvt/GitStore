# GitStore — Product Requirements Document (PRD)

**Document Version:** 1.5.0  
**Phase:** Phase 0.1.4 — Evidence Alignment, Deployment Recovery & Governance Consistency  
**Status:** Approved Product Requirements (Hardened & Reconciled)  
**Target Systems:** GitHub (Pages, Actions, Issues, Releases) & Firebase (Auth, Cloud Firestore, Security Rules, App Check)

---

## 1. Executive Summary & Vision

GitStore is a specialized, open, high-performance software discovery and distribution marketplace powered directly by GitHub and Firebase. It empowers software developers, indie hackers, and open-source creators to showcase, package, and distribute finished software products directly to users without intermediary hosting fees, centralized binary gatekeepers, or walled-garden approval taxes.

### 1.1 Core Value Proposition
- **Direct-to-User Distribution:** Visitors discover modern desktop applications, mobile packages, web apps, and developer utilities on GitStore, downloading releases directly from the creator’s verified GitHub Releases assets.
- **Zero Binary Hosting Overhead:** GitStore never re-hosts, proxies, or stores executable binaries in its own repositories or servers; all binary downloads are cryptographically and structurally verified links to authentic GitHub Release assets.
- **Strict Two-Platform Constraint:** The entire system operates exclusively on **GitHub** (Pages, Actions, Issues, Releases) and **Firebase** (Authentication, Cloud Firestore, Security Rules, App Check). No external PaaS/FaaS servers (Supabase, AWS, Vercel, Netlify) or Cloud Functions are used.
- **Static Storefront Performance:** By compiling approved submissions into static JSON catalogs served directly via GitHub Pages, storefront browsing achieves ultra-fast load times with zero operational database egress.

---

## 2. Target Audience & Personas

### Persona 1: The Indie Software Creator ("Dev Dave")
- **Profile:** Ships desktop utilities (C++, Rust, Go, Electron, Tauri, .NET), CLI tools, or mobile APKs on public GitHub repositories.
- **Pain Points:** Traditional app stores demand expensive annual developer fees, invasive code reviews, or closed distribution agreements. Direct GitHub repository pages are cluttered for non-technical end-users who cannot find the `Releases` tab or identify the correct platform asset.
- **Needs:** A clean, professional showcase link for their software, displaying screenshots, platform badges, license, and a one-click direct download button linked to their official GitHub Release.

### Persona 2: The Open-Source Software Consumer ("End-User Elena")
- **Profile:** Power user, developer, or enthusiast seeking reliable, open-source desktop software, tools, and templates.
- **Pain Points:** Shady third-party download portals inject adware, bundle malware, or provide outdated setup files.
- **Needs:** A modern, beautiful, high-contrast, searchable app store interface where each download links directly to official GitHub releases, accompanied by developer verification status, release tags, and file metadata.

### Persona 3: The GitStore Administrator ("Admin Alex")
- **Profile:** Ecosystem maintainer ensuring marketplace safety, vetting first-time submissions, handling user reports, and guarding against spam.
- **Needs:** A secure admin dashboard and automated processing pipelines that validate repository health, release tags, and asset formats without executing untrusted code or exposing administrative service account keys to the browser.

---

## 3. Product Features & Functional Requirements

### 3.1 Supported Product Formats & Globally Unique Multi-Product Repositories
- **Supported Formats:** Desktop Executables (`.exe`, `.msi`), macOS (`.dmg`), Linux (`.zip`, `.tar.gz`), Android (`.apk`), Web Apps, and Templates.
- **Multi-Product Repositories:** A single GitHub repository (monorepos, tool suites) can distribute multiple independent software applications under globally unique deterministic product IDs (`prod_{repoNumericId}_{productSlug}`) tied to the repository's immutable numeric GitHub ID (`gh_repo_{repoNumericId}`).
- **Cross-Repository Collision Prevention:** Repositories under the same owner with identical product slugs (e.g. `microsoft/toolkit-a -> viewer` vs `microsoft/toolkit-b -> viewer`) generate distinct non-colliding product IDs (`prod_892341901_viewer` vs `prod_945112002_viewer`).

### 3.2 Submission Channels & Account Reconciliation
- **Developer Web Dashboard:** Authenticated developers submit products with real-time status tracking.
- **GitHub Issue Forms:** Alternative issue-based submission route. Issue submitters do **not** receive fake placeholder accounts; submissions are recorded as `unclaimed` with their immutable numeric GitHub ID (`sub_issue_{repoNumericId}_{issueNumber}`).
- **Secure Asynchronous Claiming Queue:** Developers claim issue-submitted applications through `/claim_requests`, validated authoritatively against their linked GitHub account in Firebase Auth by the batch worker. Client creation is strictly whitelisted via `hasOnly(['requestId', 'requesterUid', 'targetSubmissionId', 'status', 'createdAt'])`, blocking all privileged and worker fields.

### 3.3 Strict Ownership, Release Authority & Verified Developer Distinction
1. **Repository Ownership:** Proves control over the repository configuration (`repo.owner.id === submitterGithubId`).
2. **Release Authority:** Direct personal repository owners possess inherent release authority to auto-publish upon passing automated asset checks. Organization members, committers, and third-party nominators lack proven release authority and are **forced to `needs_review`**.
3. **Verified Developer Identity:** A distinct platform trust badge granted by administrators based on verified track record or identity. Owning a personal repo does **not** automatically award the Verified Developer badge.
4. **Suspicious / Malformed Submissions:** Failed checks (private repo, missing asset, deleted release tag) are automatically rejected.

### 3.4 Decoupled Administrator Moderation Workflow
- Administrative operations write commands to `/moderation_requests`.
- The Actions worker validates permissions against **GitStore's canonical repository** (`Rohitpvt/GitStore`) root owner immutable numeric ID (`owner.id`) for bootstrap authorization and collaborator roles (`role_name === 'admin'` or `'maintain'`) for maintainers before applying mutations via `firebase-admin`. Zero browser credentials exist.

### 3.5 Public Storefront Capabilities
- **Homepage:** Hero showcase, featured applications, category shelves, recent releases.
- **Explore / Search Page:** Fast client-side filtering by category, platform, license, and instant search over static catalog snapshots.
- **Product Detail Page:** Deep product information, platform pills, verified developer badge (if awarded), release notes preview, direct download CTA with file size, license identifier, and external repository links.
- **Category Browsing:** Curated categories with live product count badges.

---

## 4. Platform Constraints & Boundary Enforcement

| Component | Permitted Provider | Explicitly Forbidden Providers |
|---|---|---|
| **Web Hosting & Static Delivery** | GitHub Pages | Vercel, Netlify, Cloudflare Pages, AWS S3/CloudFront |
| **Batch Worker & Privileged Automation** | GitHub Actions | AWS Lambda, Google Cloud Run, Heroku, VPS servers |
| **Source & Binary Distribution** | GitHub Releases & Public Repos | Direct file uploads to GitStore servers, S3 binary buckets |
| **Authentication Service** | Firebase Authentication | Supabase Auth, Auth0, Custom Node.js auth servers |
| **Operational Database** | Cloud Firestore | Supabase (PostgreSQL), MongoDB, DynamoDB, PlanetScale |
| **Security & Abuse Mitigation** | Firebase App Check (reCAPTCHA Enterprise) & Security Rules | Third-party WAFs, Cloudflare Turnstile, custom proxies |

---

## 5. Non-Functional Requirements & Launch Gates

### 5.1 Performance & Latency
- **Storefront Catalog Load Time:** `< 800 ms` on global connections by loading pre-compiled `data/catalog.json` from GitHub Pages.
- **Zero Operational DB Reads for Anonymous Visitors:** Anonymous browsing of the homepage, explore, and product pages makes zero direct read requests to Cloud Firestore, safeguarding the Firebase Spark free quota.

### 5.2 Security & Integrity
- **Non-Execution Invariant:** GitStore never executes submitted files or binaries. All validation is structural, cryptographic, and metadata-focused via the GitHub REST API.
- **Zero-Trust Client Security:** Firestore Security Rules prohibit client privilege self-escalation. Admin operations execute strictly inside GitHub Actions.
- **Recoverable Two-Tier Publication State Machine:** Submissions transition to `published` in Firestore only after:
  1. Compiling a non-circular manifest (`data/deployment-manifest.json` records `generatedAt` and monotonic `generationSequence`, omitting commit SHAs).
  2. Single-workflow deployment (Option A) executing `actions/upload-pages-artifact@v3` and `actions/deploy-pages@v4` in `.github/workflows/batch-processor.yml`.
  3. GitHub Pages deployment confirmation via the dedicated Pages deployment endpoint (`GET .../pages/deployments/{id}` returning `"status": "succeed"`).
  4. Public HTTP manifest verification asserting matching `generationId` and monotonic `generationSequence` evaluated against the 4-case ordering matrix.
  5. Backed by an automated reconciliation loop featuring deterministic post-commit SHA recovery via Git log inspection, and durable preservation of failed generations in `/system_deployments/{generationId}` (never prematurely reverting products to `approved`). Zero work loss guaranteed by Firestore coalescing.

### 5.3 Mandatory GitHub Pages Pre-Launch Gate
- **Pre-Launch Policy Clarification:** While password forms are eliminated to satisfy the sensitive transaction policy, formal written clarification regarding the classification of GitStore as a non-commercial open directory must be obtained from GitHub Support as a formal **Phase 4 Launch Gate**. *Classified as Unresolved / Pending External Confirmation.*

---

## 6. Success Metrics & KPIs
1. **Catalog Integrity:** 100% of published download links resolve to valid, active GitHub Releases assets.
2. **Operational Budget:** Target $0.00 / month running cost within the Firebase Spark Plan and GitHub Free tier.
3. **Processing Integrity:** Zero duplicate product records or desynchronized storefront states.
4. **Developer Trust:** 0 false-positive auto-publications of malicious or broken links via the mandatory unverified-review gate.
