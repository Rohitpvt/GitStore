# GitStore — Comprehensive Deployment & Operations Guide

**Document Version:** 1.1.0  
**Phase:** Phase 0.1 — Architecture Audit, Hardening & Final Validation  
**Status:** Approved Operations Manual (Hardened)  
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

### 2.4 Configure Firebase App Check (reCAPTCHA v3)
1. Go to **Build -> App Check**.
2. Register your Web App using **reCAPTCHA v3** (provides 1,000,000 monthly verifications free, unlike Enterprise which bills after 10k).
3. Add your authorized domains:
   - `localhost` (for local development with debug token)
   - `<your-username>.github.io` (production GitHub Pages domain)
4. Enforce App Check on **Cloud Firestore**.

### 2.5 Generate Service Account for GitHub Actions
1. In Firebase Console, open **Project Settings -> Service accounts**.
2. Click **Generate new private key**.
3. Download the resulting JSON file securely. This JSON grants administrative privileges to the GitHub Actions batch processor.

---

## 3. Step 2: GitHub Repository & Secrets Configuration

### 3.1 Repository Setup
1. Create a new public repository: `https://github.com/<your-username>/GitStore`.
2. Clone repository to your local machine:
   ```bash
   git clone https://github.com/<your-username>/GitStore.git
   cd GitStore
   ```

### 3.2 Configure GitHub Repository Secrets
1. In your GitHub repository, navigate to **Settings -> Secrets and variables -> Actions**.
2. Click **New repository secret** and add:
   - **`FIREBASE_SERVICE_ACCOUNT_JSON`**: Paste the entire raw JSON content of the Firebase service account private key generated in Step 2.5.
   - **`FIREBASE_CLIENT_CONFIG`**: Paste the JSON containing your public web app keys (apiKey, authDomain, projectId, appId).

### 3.3 Configure Administrative Collaborators
1. In your GitHub repository, navigate to **Settings -> Collaborators**.
2. Add designated platform administrators as collaborators with role **Admin** or **Maintain**.
3. The GitHub Actions batch worker uses the GitHub REST API to verify collaborator permissions before executing privileged moderation requests from the dashboard.

---

## 4. Step 3: GitHub Pages Configuration

1. In your GitHub repository, navigate to **Settings -> Pages**.
2. Under **Build and deployment**:
   - **Source:** Select **GitHub Actions**.
3. Ensure HTTPS is enforced by checking **Enforce HTTPS**.

---

## 5. Step 4: Deploying Security Rules & Database Indexes

Install the Firebase CLI locally (used for deploying rules and schema definitions):
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

Verify that `.github/workflows/batch-processor.yml` is enabled:

```yaml
name: GitStore Batch Processor

on:
  schedule:
    # Executes approximately every 10 minutes (subject to Actions queue load)
    - cron: '*/10 * * * *'
  workflow_dispatch: # Allows manual trigger from Actions tab

concurrency:
  group: gitstore-batch-worker
  cancel-in-progress: false

jobs:
  process-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Run Batch Verification & Static Compilation
        env:
          FIREBASE_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          node scripts/processor/index.js

      - name: Deploy to GitHub Pages
        id: deploy
        uses: actions/deploy-pages@v4
        if: success()

      - name: Finalize Published Status in Firestore
        if: steps.deploy.outcome == 'success'
        env:
          FIREBASE_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
        run: |
          node scripts/processor/finalize-publishing.js
```

---

## 7. Step 6: Post-Deployment Verification Checklist

Execute these checks after initial deployment:
1. **Storefront Availability:** Visit `https://<your-username>.github.io/GitStore/` and verify that the obsidian dark UI loads without console errors.
2. **Static Catalog Access:** Confirm that `https://<your-username>.github.io/GitStore/data/catalog.json` returns HTTP `200 OK`.
3. **Authentication Verification:** Click **[ CONNECT GITHUB ]** and verify that the GitHub OAuth consent screen opens and returns an authenticated user state.
4. **Submission Ingestion Test:** Submit a test open-source repository and verify that a document appears in Firestore `/submissions` with `status: 'queued'`.
5. **Batch Worker Execution:** Manually trigger `GitStore Batch Processor` under the GitHub Actions tab and confirm the submission transitions to `needs_review` or `approved`.
6. **Moderation Queue Execution:** Enqueue an approval from `admin.html` and verify the worker confirms collaborator rights, updates Firestore, and deploys static files.
