# GitStore — Product Requirements Document (PRD)

**Document Version:** 1.2.0  
**Phase:** Phase 0.1.1 — Critical Architecture Corrections & Hardening  
**Status:** Approved Product Requirements (Corrected)  
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

### 3.1 Supported Product Formats & Multi-Product Repositories
- **Supported Formats:** Desktop Executables (`.exe`, `.msi`), macOS (`.dmg`), Linux (`.zip`, `.tar.gz`), Android (`.apk`), Web Apps, and Templates.
- **Multi-Product Repositories:** A single GitHub repository (monorepos, tool suites) can distribute multiple independent software applications under distinct product IDs (`prod_{repoOwner}_{productSlug}`).

### 3.2 Submission Channels & Account Reconciliation
- **Developer Web Dashboard:** Authenticated developers submit products with real-time status tracking.
- **GitHub Issue Forms:** Alternative issue-based submission route. Issue submitters do **not** receive fake placeholder accounts; submissions are recorded as `unclaimed` with their immutable GitHub ID and claimed when the developer authenticates via GitHub OAuth.

### 3.3 Hybrid Publication Workflow & Trust Tiers
1. **Tier 1 (Direct Personal Owner):** If developer is verified and is the direct owner of the personal repository, submissions auto-publish upon passing asset checks.
2. **Tier 2 (Org Member) & Tier 3 (Committer):** Public org members or contributors do not have proven release authority. **Forced to `needs_review`.**
3. **Tier 4 (Community Nomination):** Third-party open-source nominations **never auto-publish** and are held for manual review.
4. **Suspicious / Malformed Submissions:** Failed checks (private repo, missing asset, mismatched hash) are automatically rejected.

### 3.4 Decoupled Administrator Moderation Workflow
- Administrative operations write commands to `/moderation_requests`.
- The Actions worker validates permissions against **GitStore's canonical repository** (`Rohitpvt/GitStore`) collaborator permissions (`admin` or `maintain`) before applying mutations via `firebase-admin`.

### 3.5 Public Storefront Capabilities
- **Homepage:** Hero showcase, featured applications, category shelves, recent releases.
- **Explore / Search Page:** Fast client-side filtering by category, platform, license, and instant search over static catalog snapshots.
- **Product Detail Page:** Deep product information, platform pills, verified developer badge, release notes preview, direct download CTA with file size, license identifier, and external repository links.
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
| **Security & Abuse Mitigation** | Firebase App Check & Security Rules | Third-party WAFs, Cloudflare Turnstile, custom proxies |

---

## 5. Non-Functional Requirements & Launch Gates

### 5.1 Performance & Latency
- **Storefront Catalog Load Time:** `< 800 ms` on global connections by loading pre-compiled `data/catalog.json` from GitHub Pages.
- **Zero Operational DB Reads for Anonymous Visitors:** Anonymous browsing of the homepage, explore, and product pages makes zero direct read requests to Cloud Firestore, safeguarding the Firebase Spark free quota.

### 5.2 Security & Integrity
- **Non-Execution Invariant:** GitStore never executes submitted files or binaries. All validation is structural, cryptographic, and metadata-focused via the GitHub REST API.
- **Zero-Trust Client Security:** Firestore Security Rules prohibit client privilege self-escalation. Admin operations execute strictly inside GitHub Actions.
- **Self-Healing Two-Phase Publication:** Submissions transition to `published` in Firestore only after GitHub Pages deployment confirms success, backed by an automated reconciliation loop.

### 5.3 Mandatory GitHub Pages Pre-Launch Gate
- **Pre-Launch Policy Clarification:** While password forms are eliminated to satisfy the sensitive transaction policy, formal written clarification regarding the classification of GitStore as a non-commercial open directory must be obtained from GitHub Support as a formal **Phase 4 Launch Gate**.

---

## 6. Success Metrics & KPIs
1. **Catalog Integrity:** 100% of published download links resolve to valid, active GitHub Releases assets.
2. **Operational Budget:** Target $0.00 / month running cost within the Firebase Spark Plan and GitHub Free tier.
3. **Processing Integrity:** Zero duplicate product records or desynchronized storefront states.
4. **Developer Trust:** 0 false-positive auto-publications of malicious or broken links via the mandatory unverified-review gate.
