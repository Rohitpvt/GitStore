# GitStore — Comprehensive Deployment & Operations Guide

**Document Version:** 1.5.0  
**Phase:** Phase 0.1.4 — Evidence Alignment, Deployment Recovery & Governance Consistency  
**Status:** Approved Operations Manual (Hardened & Reconciled)  
**Target Environments:** GitHub (Pages & Actions) and Firebase (Auth, Firestore, App Check)

---

## 1. Prerequisites & Account Setup

Before initiating deployment, ensure you possess:
1. A **GitHub Account / Organization** with administrative permissions to create a public repository.
2. A **Google Firebase Account** on the zero-cost **Spark Plan**.
3. Git and Node.js (v20+) installed locally for initial repository cloning and branch setup.

---

## 2. Step 1: Firebase Project Configuration

### 2.1 Create Project & Select Region
1. Navigate to the [Firebase Console](https://console.firebase.google.com/).
2. Click **Add Project**, name it `gitstore-prod`, and disable Google Analytics (optional, to keep dependencies minimal).
3. Confirm the plan is **Spark (Free $0/month)**.

### 2.2 Configure Cloud Firestore
1. In the Firebase Console, go to **Build -> Firestore Database**.
2. Click **Create Database**.
3. Select **Firestore Native Mode**.
4. Choose a region close to your primary audience (e.g., `us-central1` or `europe-west1`). Note: Region selection is permanent.
5. Choose **Start in test mode** for initial setup, which will be immediately overwritten by deploying GitStore’s verified security rules.

### 2.3 Configure Firebase Authentication
1. Go to **Build -> Authentication** and click **Get Started**.
2. **Enable GitHub Provider:**
   - Under Sign-in method, select **GitHub**.
   - Copy the provided **Authorization callback URL** (e.g., `https://gitstore-prod.firebaseapp.com/__/auth/handler`).
   - Open GitHub in a new tab: **Settings -> Developer settings -> OAuth Apps -> New OAuth App**.
   - Set Application Name to `GitStore`.
   - Set Homepage URL to `https://<your-username>.github.io/GitStore/`.
   - Paste the copied Firebase callback URL into **Authorization callback URL**.
   - Register application and generate a new **Client Secret**.
   - Paste the **Client ID** and **Client Secret** back into the Firebase GitHub Auth configuration modal and click **Save**.
3. **Enable Email Link (Passwordless):**
   - Under Sign-in method, select **Email/Password**.
   - Toggle **Enable**.
   - Toggle **Email link (passwordless sign-in)** to **ON**.
   - Click **Save**.
4. **Configure Authorized Domains:**
   - In **Authentication -> Settings -> Authorized domains**, add:
     - `localhost`
     - `127.0.0.1`
     - `<your-username>.github.io`

### 2.4 Configure Firebase App Check (reCAPTCHA Enterprise)
1. Go to **Build -> App Check**.
2. Register your Web App using **reCAPTCHA Enterprise** (`ReCaptchaEnterpriseProvider`), which provides 10,000 monthly assessments free under Google Cloud Fraud Defense.
3. Add your authorized domains:
   - `localhost` (for local development with debug token)
   - `<your-username>.github.io` (production GitHub Pages domain)
4. Enforce App Check in **monitoring mode initially**, moving to strict enforcement in Phase 6.

### 2.5 Generate Service Account for GitHub Actions
1. In Firebase Console, open **Project Settings -> Service accounts**.
2. Click **Generate new private key**.
3. Download the resulting JSON file securely. This JSON grants administrative privileges to the GitHub Actions batch processor.

---

## 3. Step 2: GitHub Repository & Secrets Configuration

### 3.1 Repository Setup
1. Create a new public repository: `https://github.com/Rohitpvt/GitStore`.
2. Clone repository to your local machine:
   ```bash
   git clone https://github.com/Rohitpvt/GitStore.git
   cd GitStore
   ```

### 3.2 Configure GitHub Repository Secrets
1. In your GitHub repository, navigate to **Settings -> Secrets and variables -> Actions**.
2. Click **New repository secret** and add:
   - **`FIREBASE_SERVICE_ACCOUNT_JSON`**: Paste the entire raw JSON content of the Firebase service account private key generated in Step 2.5.
   - **`FIREBASE_CLIENT_CONFIG`**: Paste the JSON containing your public web app keys (apiKey, authDomain, projectId, appId).
   - **`ADMIN_VERIFY_TOKEN`**: A Fine-Grained Personal Access Token with repository `Metadata: read` permission on `Rohitpvt/GitStore`. (Used as a fallback if the default `GITHUB_TOKEN` is denied access to collaborator permission endpoints with HTTP 403. *Classified as PENDING INTEGRATION TEST*).

### 3.3 Configure Canonical GitStore Collaborators & Root Administrator
1. In your GitHub repository (`Rohitpvt/GitStore`), navigate to **Settings -> Collaborators**.
2. The repository owner (`Rohitpvt`) is verified automatically as the root administrator by matching their linked immutable numeric GitHub ID (`owner.id`) from repository metadata.
3. Add designated platform administrators as collaborators with role **Admin** or **Maintain**.
4. The GitHub Actions batch worker uses the GitHub REST API (`GET /repos/Rohitpvt/GitStore/collaborators/{username}/permission`) to verify collaborator permissions before executing privileged moderation requests from `/moderation_requests`.
5. Note that GitHub maps the `maintain` role to `write` in its legacy `permission` field; the worker checks `role_name === 'admin' || role_name === 'maintain'`.

---

## 4. Step 3: GitHub Pages Configuration & Publication Verification

1. In your GitHub repository, navigate to **Settings -> Pages**.
2. Under **Build and deployment**:
   - **Source:** Select **GitHub Actions**.
3. Ensure HTTPS is enforced by checking **Enforce HTTPS**.
4. Publication Verification Architecture:
   - Modern deployments use `actions/upload-pages-artifact@v3` and `actions/deploy-pages@v4` targeting the `github-pages` environment within the same trusted workflow run (Option A).
   - The batch worker generates `data/deployment-manifest.json` containing `{ generationId, generationSequence, generatedAt, productCount, schemaVersion: "1.5.0", environment: "github-pages" }` (omitting the Git commit SHA, and accurately recording generation time; deployment timestamps are recorded in Firestore).
   - The worker commits to `main`, captures `catalogCommitSha` via `git rev-parse HEAD`, records the generation in Firestore `/system_deployments/{generationId}`, and queries the dedicated GitHub Pages deployment endpoint `GET /repos/Rohitpvt/GitStore/pages/deployments/{pages_deployment_id}` for status `"succeed"`.
   - In case of a worker crash post-push before Firestore update, the startup reconciler recovers the SHA via `git log -n 10` and `git show` blob inspection with disambiguation.
   - Finally, the worker performs a public HTTP check on `https://rohitpvt.github.io/GitStore/data/deployment-manifest.json` confirming `generationId` and monotonic `generationSequence` before finalizing database status to `published`.

---

## 5. Step 4: Deploying Security Rules & Database Indexes

Install the Firebase CLI locally:
```bash
npm install -g firebase-tools
firebase login
firebase use --add gitstore-prod
```

Deploy the security rules and composite indexes:
```bash
# Deploy hardened Firestore security rules
firebase deploy --only firestore:rules

# Deploy Firestore composite indexes
firebase deploy --only firestore:indexes
```

---

## 6. Step 5: Automating GitHub Actions Batch Processor

Verify that `.github/workflows/batch-processor.yml` is enabled with least privilege permissions and environment concurrency:

```yaml
name: GitStore Batch Processor

on:
  schedule:
    # Executes approximately every 10 minutes (subject to Actions platform queue load)
    - cron: '*/10 * * * *'
  workflow_dispatch: # Allows manual trigger from Actions tab by collaborators

permissions:
  contents: write        # Required for committing static catalog and manifest
  pages: write           # Required for minting GitHub Pages deployment artifacts
  id-token: write        # Required for OpenID Connect (OIDC) token minting
  metadata: read         # Required for inspecting collaborator permissions

concurrency:
  group: 'github-pages'
  cancel-in-progress: false

jobs:
  build-commit-and-deploy:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: github-pages
      url: ${{ steps.deploy.outputs.page_url }}
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Run Reconciliation, Verification & Static Catalog Compilation
        env:
          FIREBASE_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ADMIN_VERIFY_TOKEN: ${{ secrets.ADMIN_VERIFY_TOKEN }}
          CANONICAL_REPOSITORY: ${{ github.repository }}
        run: |
          node scripts/processor/index.js

      - name: Commit & Push Catalog to Main
        run: |
          git config user.name "gitstore-bot"
          git config user.email "bot@gitstore.dev"
          git add data/
          git diff-index --quiet HEAD || git commit -m "chore(catalog): publish static catalog [skip ci]"
          git push origin main
          echo "CATALOG_SHA=$(git rev-parse HEAD)" >> $GITHUB_ENV

      - name: Package Pages Artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: .

      - name: Deploy to GitHub Pages (Option A: Same Trusted Run)
        id: deploy
        uses: actions/deploy-pages@v4

      - name: Finalize Published Status in Firestore
        if: steps.deploy.outcome == 'success'
        env:
          FIREBASE_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          CATALOG_SHA: ${{ env.CATALOG_SHA }}
        run: |
          node scripts/processor/finalize-publishing.js
```

*(Evidence Classification: Workflow design and action versions are DOCUMENTATION_VERIFIED; live execution on the repository is PENDING_INTEGRATION_TEST).*

---

## 7. Step 6: Post-Deployment Verification Checklist

Execute these checks after initial deployment:
1. **Storefront Availability:** Visit `https://rohitpvt.github.io/GitStore/` and verify that the obsidian dark UI loads without console errors.
2. **Static Catalog Access:** Confirm that `https://rohitpvt.github.io/GitStore/data/catalog.json` returns HTTP `200 OK`.
3. **Authentication Verification:** Click **[ CONNECT GITHUB ]** and verify that the GitHub OAuth consent screen opens and returns an authenticated user state.
4. **Submission Ingestion Test:** Submit a test open-source repository and verify that a document appears in Firestore `/submissions` with `status: 'queued'`.
5. **Batch Worker Execution:** Manually trigger `GitStore Batch Processor` under the GitHub Actions tab and confirm the submission transitions to `needs_review` or `approved`.
6. **Moderation Queue Execution:** Enqueue an approval from `admin.html` and verify the worker confirms collaborator rights on `Rohitpvt/GitStore`, updates Firestore, and deploys static files.
