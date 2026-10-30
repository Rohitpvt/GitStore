# GitStore — Product Requirements Document (PRD)

**Document Version:** 1.1.0  
**Phase:** Phase 0.1 — Architecture Audit, Hardening & Final Validation  
**Status:** Approved Product Requirements (Hardened)  
**Target Systems:** GitHub (Pages, Actions, Issues, Releases) & Firebase (Auth, Cloud Firestore, Security Rules, App Check)

---

## 1. Executive Summary & Vision

GitStore is a specialized, open, high-performance software discovery and distribution marketplace powered directly by GitHub and Firebase. It empowers software developers, indie hackers, and open-source creators to showcase, package, and distribute finished software products directly to users without intermediary hosting fees, centralized binary gatekeepers, or walled-garden approval taxes.

### 1.1 Core Value Proposition
- **Direct-to-User Distribution:** Visitors discover modern desktop applications, mobile packages, web apps, and developer utilities on GitStore, downloading releases directly from the creator’s verified GitHub Releases assets.
- **Zero Binary Hosting Overhead:** GitStore never re-hosts, proxies, or stores executable binaries in its own repositories or servers; all binary downloads are cryptographically and structurally verified links to authentic GitHub Release assets.
- **Strict Two-Platform Constraint:** The entire system operates exclusively on **GitHub** (Pages, Actions, Issues, Releases) and **Firebase** (Authentication, Cloud Firestore, Security Rules, App Check). No external PaaS/FaaS servers (Supabase, AWS, Vercel, Netlify) or Cloud Functions are used.
- **Static Storefront Performance:** By compiling approved submissions into static JSON catalogs served over GitHub's global Fastly CDN via GitHub Pages, storefront browsing achieves ultra-fast load times with near-zero operational database egress.

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

### 3.1 Supported Product Formats
GitStore distributes finished products across five major digital formats:
1. **Desktop Executables & Installers:** Windows (`.exe`, `.msi`), macOS (`.dmg`), Linux (`.zip`, `.tar.gz`).
2. **Mobile Packages:** Android (`.apk`).
3. **Compressed Application Archives:** Universal binaries and portable suites (`.zip`, `.tar.gz`).
4. **Web Applications & SaaS Previews:** Hosted web applications with verified public repository sources and live demo URLs.
5. **Developer Utilities & Website Templates:** Source packages and starter themes distributed via GitHub Releases.

### 3.2 Submission Channels & Ownership Verification
The system accommodates two distinct submission channels feeding into an identical processing queue:
- **Developer Web Dashboard:** Authenticated web portal allowing developers to connect their GitHub account, paste their public repository URL, select release tags and assets, specify metadata, and track submission states.
- **GitHub Issue Forms:** An alternative, issue-based submission route using structured `.github/ISSUE_TEMPLATE/product-submission.yml`.
- **Repository Ownership Framework:**
  - Submissions undergo automated ownership checks: direct repository owner, public organization member, or active code committer.
  - Submissions failing ownership proof are categorized as `community_nomination`.

### 3.3 Hybrid Publication Workflow & Trust Gates
To prevent spam, abuse, and repository hijacking:
1. **Unverified Developers & Community Nominations:** Must pass automated verification AND mandatory administrator review. They transition to `needs_review`.
2. **Verified Developers (Author Submissions Only):** Submissions from developers marked verified with proven repository ownership auto-publish immediately upon passing automated repository and asset validation.
3. **Community Nominations Restricted:** Community nominations **never auto-publish**, regardless of the nominator's verification status.
4. **Suspicious / Malformed Submissions:** Any submission with failing checks (private repo, missing asset, mismatched hash/extension) is automatically rejected.

### 3.4 Decoupled Administrator Moderation Workflow
- Administrative operations (approving submissions, rejecting products, verifying developers, delisting) are initiated from the Admin Dashboard by writing to an asynchronous `/moderation_requests` command queue.
- The GitHub Actions batch worker validates the administrator's repository permissions (`admin` or `maintain`) via GitHub REST API, executes the mutation via `firebase-admin`, records an immutable audit log, recompiles the static catalog, and deploys to GitHub Pages.

### 3.5 Public Storefront Capabilities
- **Homepage:** Hero showcase, featured applications, category shelves, recent releases, and ecosystem metrics.
- **Explore / Search Page:** Fast client-side filtering by category, platform (Windows, macOS, Linux, Android, Web), license, and instant search over static catalog snapshots.
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
- **Storefront Catalog Load Time:** `< 800 ms` on global connections by loading pre-compiled `data/catalog.json` from GitHub Pages CDN.
- **Zero Operational DB Reads for Anonymous Visitors:** Anonymous browsing of the homepage, explore, and product pages makes zero direct read requests to Cloud Firestore, safeguarding the Firebase Spark free quota.

### 5.2 Security & Integrity
- **Non-Execution Invariant:** GitStore never executes submitted files or binaries. All validation is structural, cryptographic, and metadata-focused via the GitHub REST API.
- **Zero-Trust Client Security:** Firestore Security Rules prohibit client privilege self-escalation. Admin operations execute strictly inside GitHub Actions.
- **Two-Phase Publication:** Submissions transition to `published` in Firestore only after GitHub Pages deployment confirms success.

### 5.3 Mandatory GitHub Pages Pre-Launch Gate
- **Pre-Launch Policy Clarification:** While password forms are eliminated to satisfy the sensitive transaction policy, formal written clarification regarding the classification of GitStore as a non-commercial open directory must be obtained from GitHub Support as a formal **Phase 4 Launch Gate**.

---

## 6. Success Metrics & KPIs
1. **Catalog Integrity:** 100% of published download links resolve to valid, active GitHub Releases assets.
2. **Operational Budget:** Target $0.00 / month running cost within the Firebase Spark Plan and GitHub Free tier.
3. **Processing Integrity:** Zero duplicate product records or desynchronized storefront states.
4. **Developer Trust:** 0 false-positive auto-publications of malicious or broken links via the mandatory unverified-review gate.
