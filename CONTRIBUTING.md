# Contributing to GitStore

Thank you for your interest in contributing to **GitStore**!

GitStore is a specialized, open, high-performance software discovery and distribution marketplace powered directly by **GitHub** and **Firebase**.

---

## 🏛️ Architectural Guardrails

Before contributing, please review our core architectural invariants:

1. **Two External Platforms Only:** GitStore strictly operates on **GitHub** (Pages, Actions, Issues, Releases) and **Firebase** (Auth, Firestore, Security Rules, App Check). Do not submit PRs introducing AWS, Supabase, Vercel, Netlify, or third-party servers.
2. **Framework-Free Frontend:** The frontend is built with **HTML5, CSS3, and Vanilla JavaScript (ES6 Modules)**. PRs introducing React, Vue, Next.js, or complex bundler toolchains will not be accepted.
3. **Binary Non-Execution:** GitStore never re-hosts or executes third-party binaries. All download buttons link directly to official upstream GitHub Releases assets.
4. **Phase Control:** All active work must adhere to the current phase documented in [`PHASE_PLAN.md`](PHASE_PLAN.md).

---

## 📦 How to Submit a Software Product to GitStore

There are two official methods to submit an application:

### Option A: Developer Dashboard (Recommended)
Visit the live GitStore web application, authenticate with GitHub, and use the interactive **Submit App** wizard to provide your repository URL, release tag, and asset selection.

### Option B: GitHub Issue Form
Open a new issue using our structured [Product Submission Issue Template](.github/ISSUE_TEMPLATE/product-submission.yml). Our automated GitHub Actions batch processor will verify your repository and release assets.

---

## 💻 Code Contributions

### 1. Setting Up Local Development
1. Fork the repository and clone it locally:
   ```bash
   git clone https://github.com/<your-username>/GitStore.git
   cd GitStore
   ```
2. Because GitStore uses Vanilla JavaScript without bundlers, you can serve the project using any local HTTP static server:
   ```bash
   # Python 3
   python -m http.server 8000
   
   # Or Node.js http-server / serve
   npx serve .
   ```
3. Open `http://localhost:8000` in your browser.

### 2. Commit Message Guidelines
We follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:
- `feat(catalog): add category filter drawer`
- `fix(auth): handle account linking error state`
- `docs(schema): update moderation_requests definition`
- `chore(deps): update test fixtures`

### 3. Pull Request Process
1. Ensure your branch is based on `main` and up to date.
2. Verify that your changes adhere to the visual standards in [`docs/UI_DESIGN_SYSTEM.md`](docs/UI_DESIGN_SYSTEM.md).
3. Confirm that all automated tests pass.
4. Open a Pull Request referencing any relevant issues.

---

## 📜 Code of Conduct
All contributors and maintainers are expected to adhere to our [Code of Conduct](CODE_OF_CONDUCT.md).
