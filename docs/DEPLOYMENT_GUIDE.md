# GitStore — Comprehensive Deployment & Operations Guide

**Document Version:** 1.8.0  
**Phase:** Phase 0.1.7 — Replay Protection, Firestore Field Integrity & Deployment State Consistency  
**Status:** IN_REVIEW — Awaiting Explicit User Approval  
**Target Environments:** GitHub (Pages & Actions) and Firebase (Auth, Firestore, App Check)

---

## 1. Prerequisites & Account Setup

Before initiating deployment, ensure you possess:
1. A **GitHub Account / Organization** with administrative permissions to create a public repository.
2. A **Google Firebase Account** configured to operate on the zero-cost **Spark Plan** without an attached billing account.
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
5. Deploy GitStore’s verified security rules from `docs/DATABASE_SCHEMA.md` to ensure private user documents, internal collections, and `/consumed_stepup_nonces` are locked down immediately.

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
6. For every administrative action, the frontend requires interactive **fresh GitHub reauthentication with action-bound proof (Option A)**, producing a client data-integrity checksum (`actionHash`) and single-use `actionNonce` recorded and leased atomically in `/consumed_stepup_nonces/{nonceHash}` by the worker.

---

## 4. Step 3: GitHub Pages Configuration & Publication Verification

1. In your GitHub repository, navigate to **Settings -> Pages**.
2. Under **Build and deployment**:
   - **Source:** Select **GitHub Actions**.
3. Ensure HTTPS is enforced by checking **Enforce HTTPS**.
4. Publication Verification Architecture:
   - Deployments use the **least-privilege split-job architecture** (`build-and-commit` and `deploy-to-pages`) targeting the `github-pages` environment within the same trusted workflow run (Option A).
   - The batch worker generates `data/deployment-manifest.json` containing `{ generationId, generationSequence, generatedAt, catalogDigest, productCount, schemaVersion: "1.8.0", environment: "github-pages" }`.
   - The worker commits to `main`, captures `catalogCommitSha` via `git rev-parse HEAD`, records the generation in Firestore `/system_deployments/{generationId}`, and queries the official GitHub Pages deployment endpoint `GET /repos/Rohitpvt/GitStore/pages/deployments/{pages_deployment_id}` (where the parameter accepts a numeric deployment ID or string commit SHA) for status `"succeed"`.
   - Status interpretation strictly adheres to official GitHub Pages documentation:
     - `status === 'succeed'`: Deployment succeeded and is live.
     - `status === 'queued' || status === 'in_progress'`: Deployment is currently building; worker waits and polls with exponential backoff.
     - `status === 'errored'`: Deployment failed on GitHub Pages infrastructure.
     - `HTTP 404`: No deployment exists for this SHA/ID (triggers Case B in recovery reconciler).
     - `HTTP 401 / 403`: Workflow permissions defect. Worker halts immediately.
     - `HTTP 5xx`: Transient GitHub outage. Worker preserves pending state for next cycle.
     - Any unrecognized or unexpected status string is rejected and halts execution.
   - In case of a worker crash post-push before Firestore update, the startup reconciler recovers the SHA via complete pagination of `origin/main` commits following `Link: rel="next"`, proves branch ancestry via `git merge-base --is-ancestor`, and matches the recursive canonical `catalogDigest`.
   - In Case B (commit exists on remote, but no Pages deployment exists), the reconciler stages `site/`, returns `{ deployNeeded: true, generationId, generationSequence, catalogCommitSha, catalogDigest }`, and sets `deploy_needed=true` in `$GITHUB_OUTPUT`. Job 1 uploads the Pages artifact, and Job 2 runs conditionally to execute the pre-deployment guard and `actions/deploy-pages@v4`.
   - Public HTTP checks on `https://rohitpvt.github.io/GitStore/data/deployment-manifest.json` confirm `generationId`, `generationSequence`, and `catalogDigest` before finalizing database status to `published` and advancing durable revisions in `/system_config/catalog_state` via an atomic transaction. If `capturedRevision < currentPublishedRevision`, the transaction aborts with `stale_aborted` to prevent rollbacks.

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

## 6. Step 5: Automating Least-Privilege Split-Job GitHub Actions Batch Processor

Verify that `.github/workflows/batch-processor.yml` is enabled with least-privilege job permissions, explicit cross-job outputs, isolated staging into `site/`, and the pre-deployment 4-case ordering guard:

```yaml
name: GitStore Batch Processor

on:
  schedule:
    # Executes approximately every 10 minutes (subject to Actions platform queue load)
    - cron: '*/10 * * * *'
  workflow_dispatch: # Allows manual trigger from Actions tab by collaborators

concurrency:
  group: 'github-pages'
  cancel-in-progress: false

jobs:
  # ---------------------------------------------------------------------------
  # Job 1: Build, Validate, Compile & Commit Static Catalog Assets
  # ---------------------------------------------------------------------------
  build-and-commit:
    runs-on: ubuntu-latest
    if: github.repository == 'Rohitpvt/GitStore' && github.ref == 'refs/heads/main'
    permissions:
      contents: write        # Required for git push of compiled static catalog and manifest
      metadata: read         # Required for inspecting repository metadata and permissions
    outputs:
      generation_id: ${{ steps.catalog.outputs.generation_id }}
      generation_sequence: ${{ steps.catalog.outputs.generation_sequence }}
      catalog_commit_sha: ${{ steps.commit.outputs.commit_sha || steps.catalog.outputs.catalog_commit_sha }}
      catalog_digest: ${{ steps.catalog.outputs.catalog_digest }}
      deploy_needed: ${{ steps.catalog.outputs.deploy_needed }}
    steps:
      - name: Checkout Repository (Unshallow)
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Setup Node.js Runtime
        uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5a1 # v4.1.0
        with:
          node-version: 24

      - name: Reconcile, Validate Submissions & Compile Catalog
        id: catalog
        env:
          FIREBASE_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ADMIN_VERIFY_TOKEN: ${{ secrets.ADMIN_VERIFY_TOKEN }}
          CANONICAL_REPOSITORY: ${{ github.repository }}
        run: |
          node scripts/processor/index.js

      - name: Stage Public Files into Isolated site/ Directory
        if: steps.catalog.outputs.deploy_needed == 'true'
        run: |
          mkdir -p site/data site/assets
          cp index.html admin.html dashboard.html submit.html product.html site/
          cp -r assets/* site/assets/
          cp -r data/* site/data/

      - name: Commit & Push Catalog to Main
        id: commit
        if: steps.catalog.outputs.deploy_needed == 'true'
        run: |
          git config user.name "gitstore-bot"
          git config user.email "bot@gitstore.dev"
          git add data/
          git diff-index --quiet HEAD || git commit -m "chore(catalog): publish static catalog [skip ci]"
          git push origin main
          echo "commit_sha=$(git rev-parse HEAD)" >> $GITHUB_OUTPUT

      - name: Upload Pages Staging Artifact
        if: steps.catalog.outputs.deploy_needed == 'true'
        uses: actions/upload-pages-artifact@56afc609e74202658d3ffba0e8f6dda462b719fa # v3.0.1
        with:
          path: site/

  # ---------------------------------------------------------------------------
  # Job 2: Deploy to GitHub Pages & Reconcile Firestore Database
  # ---------------------------------------------------------------------------
  deploy-to-pages:
    needs: build-and-commit
    runs-on: ubuntu-latest
    if: github.repository == 'Rohitpvt/GitStore' && github.ref == 'refs/heads/main' && needs.build-and-commit.outputs.deploy_needed == 'true'
    permissions:
      pages: write           # Required for GitHub Pages deployment
      id-token: write        # Required for OpenID Connect (OIDC) token minting
      contents: read         # Read-only repository access
      metadata: read         # Repository metadata access
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Checkout Repository for Guard Scripts
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 1

      - name: Setup Node.js Runtime
        uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5a1 # v4.1.0
        with:
          node-version: 24

      - name: Pre-Deployment Generation Ordering Guard (Shared Decision Function)
        env:
          FIREBASE_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          GENERATION_SEQUENCE: ${{ needs.build-and-commit.outputs.generation_sequence }}
          GENERATION_ID: ${{ needs.build-and-commit.outputs.generation_id }}
        run: |
          node scripts/processor/pre-deploy-guard.js

      - name: Deploy Artifact to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@d6db90164ac5ed86f2b6aed7e0febac5b3c0c03e # v4.0.5

      - name: Reconcile & Finalize Firestore Status
        if: steps.deployment.outcome == 'success'
        env:
          FIREBASE_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GENERATION_ID: ${{ needs.build-and-commit.outputs.generation_id }}
          CATALOG_COMMIT_SHA: ${{ needs.build-and-commit.outputs.catalog_commit_sha }}
          CATALOG_DIGEST: ${{ needs.build-and-commit.outputs.catalog_digest }}
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
4. **Submission Ingestion Test:** Submit a test open-source repository and verify that a document appears in Firestore `/submissions` with `status: 'queued'`. Verify that the client is prevented by security rules from setting any worker-managed fields (`validationResults`, `ownershipProof`, `processingLease`, `adminNotes`, `reviewedBy`, `reviewedAt`, `publishedProductId`).
5. **Batch Worker Execution:** Manually trigger `GitStore Batch Processor` under the GitHub Actions tab and confirm the submission transitions to `needs_review` or `approved`.
6. **Step-Up Moderation Execution:** Enqueue an approval from `admin.html` with interactive step-up reauthentication. Verify that:
   - The client computes `actionHash` (data-integrity checksum) and generates single-use `actionNonce`.
   - The worker atomically claims the moderation request AND inserts `/consumed_stepup_nonces/{nonceHash}`.
   - Any replay of the same nonce is rejected immediately with `NONCE_ALREADY_CONSUMED`.
   - The worker validates `targetDocumentPath` against canonical action mappings.
   - The worker validates GitHub collaborator rights on `Rohitpvt/GitStore` (`role_name === 'admin' || role_name === 'maintain'`).
   - The durable catalog revision transaction executes atomically and aborts if `capturedRevision < currentPublishedRevision`.
   - The static catalog compiles, stages to `site/`, commits, uploads Pages artifact, passes the pre-deploy ordering guard, and deploys via `actions/deploy-pages@v4`.
