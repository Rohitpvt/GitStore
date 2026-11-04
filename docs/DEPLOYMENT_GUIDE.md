# GitStore — Comprehensive Deployment & Operations Guide

**Document Version:** 1.11.0  
**Phase:** Phase 0.1.11 — Fan-Out Generation Fencing, Idempotent Retry Accounting & Schema Truthfulness  
**Status:** IN_REVIEW — Awaiting Explicit User Approval  
**Target Environments:** GitHub (Pages & Actions) and Firebase (Auth, Firestore, App Check)

---

## 1. Prerequisites & Account Setup

Before initiating deployment, ensure you possess:
1. A **GitHub Account / Organization** with administrative permissions to create a public repository.
2. A **Google Firebase Account** configured to operate on the zero-cost **Spark Plan** without an attached billing account.
3. Git and Node.js (v24+) installed locally for initial repository cloning and branch setup.

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
5. Deploy GitStore’s verified security rules from `docs/DATABASE_SCHEMA.md` to ensure private user documents, internal collections, `/consumed_stepup_nonces`, and action-specific moderation payloads are locked down immediately.

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
5. For every administrative action, the frontend requires interactive **fresh GitHub reauthentication with action-bound proof (Option A)**, producing a client data-integrity checksum (`actionHash`), single-use `actionNonce` recorded and leased atomically in `/consumed_stepup_nonces/{nonceHash}` by the worker, and an action-specific validated payload.

---

## 4. Step 3: GitHub Pages Configuration & Publication Verification

1. In your GitHub repository, navigate to **Settings -> Pages**.
2. Under **Build and deployment**:
   - **Source:** Select **GitHub Actions**.
3. Ensure HTTPS is enforced by checking **Enforce HTTPS**.
4. Publication Verification Architecture:
   - Deployments use the **least-privilege split-job architecture** (`build-and-commit` and `deploy-to-pages`) targeting the `github-pages` environment within the same trusted workflow run (Option A).
   - Actions are pinned to verified Node 24 releases using 40-character commit SHAs.
   - The batch worker generates `data/deployment-manifest.json` containing `{ generationId, generationSequence, generatedAt, catalogDigest, productCount, schemaVersion: "1.9.0", environment: "github-pages" }`.
   - Single Authoritative Staging: The `site/` staging directory is constructed authoritatively by the processor script from the exact target commit. The digest of `site/data` is recomputed and verified (`stagedDigest === catalogDigest`). No post-staging copy steps occur in the workflow.
   - Step outputs are exported via direct append writes to `$GITHUB_OUTPUT`.
   - The worker commits to `main`, captures `catalogCommitSha` via `git rev-parse HEAD`, records the generation in Firestore `/system_deployments/{generationId}`, and queries the official dedicated GitHub Pages deployment endpoint `GET /repos/Rohitpvt/GitStore/pages/deployments/{pages_deployment_id}` for status `"status": "succeed"`.
   - In case of a worker crash post-push before Firestore update, the startup reconciler recovers the SHA via complete pagination of `origin/main` commits following `Link: rel="next"`, proves branch ancestry via `git merge-base --is-ancestor`, and matches the recursive canonical `catalogDigest`. Granular error classification prevents network errors or rate limits from triggering false empty searches.
   - In Case B (commit exists on remote, but no Pages deployment exists), the reconciler stages `site/` from the target commit, verifies `stagedDigest`, returns `{ deployNeeded: true, generationId, generationSequence, catalogCommitSha, catalogDigest, capturedRevision }`, and sets `deploy_needed=true` in `$GITHUB_OUTPUT`. Job 1 uploads the Pages artifact, and Job 2 runs conditionally to execute the pre-deployment guard and `actions/deploy-pages@v5.0.1`.
   - Public HTTP checks on `https://rohitpvt.github.io/GitStore/data/deployment-manifest.json` confirm `generationId`, `generationSequence`, and `catalogDigest`.
   - 2-Tier Transactional Finalization & Bounded Subcollections: An atomic transaction reads `/system_config/catalog_state`, `/system_config/deployment_state`, and the generation record BEFORE any writes. Inside the transaction, it evaluates `evaluateGenerationOrdering` and catalog revision monotonicity. If `capturedRevision < currentPublishedRevision`, the transaction aborts with `stale_aborted` to prevent rollbacks. It commits the global publication anchor in 3 writes, while submission fan-out work is partitioned into bounded deterministic subcollections (`/system_deployments/{generationId}/fanout_batches/{batchId}`) capped at 400 IDs each (~13 KB << 1 MiB limit). Parent documents store only $O(1)$ scalar summary counters. Fan-out executes conditional transactional updates enforcing status eligibility (`approved` only; never resurrecting `rejected`, `delisted`, or `withdrawn` submissions) and strictly monotonic sequence advancement (`publishedGenerationSequence`), preventing stale fan-out overwrites across all concurrency interleavings.

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

Verify that `.github/workflows/batch-processor.yml` is enabled with least-privilege job permissions, explicit cross-job outputs, clean staging into `site/`, verified Node 24 action pins, and the pre-deployment shared ordering guard:

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
  # Job 1: Build, Validate, Compile & Commit Static Catalog Assets (Node 24)
  # ---------------------------------------------------------------------------
  build-and-commit:
    runs-on: ubuntu-latest
    if: github.repository == 'Rohitpvt/GitStore' && github.ref == 'refs/heads/main'
    permissions:
      contents: write        # Required for git push of compiled static catalog and manifest
    outputs:
      generation_id: ${{ steps.catalog.outputs.generation_id }}
      generation_sequence: ${{ steps.catalog.outputs.generation_sequence }}
      catalog_commit_sha: ${{ steps.resolve-sha.outputs.deploy_commit_sha }}
      catalog_digest: ${{ steps.catalog.outputs.catalog_digest }}
      captured_revision: ${{ steps.catalog.outputs.captured_revision }}
      deploy_needed: ${{ steps.catalog.outputs.deploy_needed }}
      commit_needed: ${{ steps.catalog.outputs.commit_needed }}
    steps:
      - name: Checkout Repository (Unshallow)
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1 (Node 24)
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Setup Node.js Runtime
        uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0 (Node 24)
        with:
          node-version: 24

      - name: Reconcile, Validate Submissions & Authoritatively Stage Catalog
        id: catalog
        env:
          FIREBASE_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ADMIN_VERIFY_TOKEN: ${{ secrets.ADMIN_VERIFY_TOKEN }}
          CANONICAL_REPOSITORY: ${{ github.repository }}
        run: |
          node scripts/processor/index.js

      - name: Commit & Push Catalog to Main
        id: commit
        if: steps.catalog.outputs.commit_needed == 'true'
        run: |
          git config user.name "gitstore-bot"
          git config user.email "bot@gitstore.dev"
          git add data/
          git diff-index --quiet HEAD || git commit -m "chore(catalog): publish static catalog [skip ci]"
          git push origin main
          NEW_SHA=$(git rev-parse HEAD)
          echo "new_commit_sha=${NEW_SHA}" >> $GITHUB_OUTPUT

      - name: Resolve Deployment Commit SHA
        id: resolve-sha
        if: steps.catalog.outputs.deploy_needed == 'true'
        run: |
          if [ "${{ steps.catalog.outputs.commit_needed }}" = "true" ]; then
            DEPLOY_SHA="${{ steps.commit.outputs.new_commit_sha }}"
            if [ -z "$DEPLOY_SHA" ] || [ ${#DEPLOY_SHA} -ne 40 ]; then
              echo "::error::New generation commit SHA is invalid or empty!"
              exit 1
            fi
            echo "deploy_commit_sha=${DEPLOY_SHA}" >> $GITHUB_OUTPUT
          else
            DEPLOY_SHA="${{ steps.catalog.outputs.catalog_commit_sha }}"
            if [ -z "$DEPLOY_SHA" ] || [ ${#DEPLOY_SHA} -ne 40 ]; then
              echo "::error::Case B recovered commit SHA is invalid or empty!"
              exit 1
            fi
            echo "deploy_commit_sha=${DEPLOY_SHA}" >> $GITHUB_OUTPUT
          fi

      - name: Upload Pages Staging Artifact
        if: steps.catalog.outputs.deploy_needed == 'true'
        uses: actions/upload-pages-artifact@fc324d3547104276b827a68afc52ff2a11cc49c9 # v5.0.0 (Composite / Node 24 Compatible)
        with:
          path: site/

  # ---------------------------------------------------------------------------
  # Job 2: Deploy to GitHub Pages & Reconcile Firestore Database (Node 24)
  # ---------------------------------------------------------------------------
  deploy-to-pages:
    needs: build-and-commit
    runs-on: ubuntu-latest
    if: github.repository == 'Rohitpvt/GitStore' && github.ref == 'refs/heads/main' && needs.build-and-commit.outputs.deploy_needed == 'true'
    permissions:
      pages: write           # Required for GitHub Pages deployment
      id-token: write        # Required for OpenID Connect (OIDC) token minting
      contents: read         # Read-only repository access
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Checkout Repository for Guard Scripts
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1 (Node 24)
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 1

      - name: Setup Node.js Runtime
        uses: actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0 (Node 24)
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
        uses: actions/deploy-pages@368f82528645a54fb793d4d04e342629a3f51346 # v5.0.1 (Node 24)

      - name: Reconcile & Finalize Firestore Status (2-Tier Transaction)
        if: steps.deployment.outcome == 'success'
        env:
          FIREBASE_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GENERATION_ID: ${{ needs.build-and-commit.outputs.generation_id }}
          CATALOG_COMMIT_SHA: ${{ needs.build-and-commit.outputs.catalog_commit_sha }}
          CATALOG_DIGEST: ${{ needs.build-and-commit.outputs.catalog_digest }}
          CAPTURED_REVISION: ${{ needs.build-and-commit.outputs.captured_revision }}
        run: |
          node scripts/processor/finalize-publishing.js
```

*(Evidence Classification: Workflow design and action versions are DOCUMENTATION_VERIFIED; live execution on the repository is PENDING_INTEGRATION_TEST).*

### 6.1 Official GitHub Actions Permission Reference

GitHub Actions enforces strict least-privilege token access via the top-level or job-level `permissions:` map. Unsupported permission keys are rejected during workflow schema validation:

| Permission Key | Value | Purpose | Assigned Job | Official Documentation Source |
|---|---|---|---|---|
| `contents` | `write` | Pushes compiled catalog JSON and deployment manifest to `main`. | `build-and-commit` | GitHub Docs: Automatic token authentication / Permissions for the GITHUB_TOKEN |
| `pages` | `write` | Creates and deploys GitHub Pages release packages via Pages API. | `deploy-to-pages` | GitHub Docs: Using environments for deployment / Pages permissions |
| `id-token` | `write` | Mints OIDC JWT for GitHub Pages deployment service authentication. | `deploy-to-pages` | GitHub Docs: OpenID Connect in GitHub Actions |
| `contents` | `read` | Read-only repository checkout for guard scripts. | `deploy-to-pages` | GitHub Docs: Permissions for the GITHUB_TOKEN |

> [!IMPORTANT]
> **Resolution of `metadata: read`:** GitHub's configurable `GITHUB_TOKEN` permission schema does **not** include `metadata` as a configurable key. In GitHub Actions, read access to repository metadata (commits, refs, and repository properties) is implicitly and unconditionally granted to `GITHUB_TOKEN` by default. Attempting to explicitly specify `metadata: read` in the `permissions:` block causes workflow parser failure with `Unexpected value 'metadata'`. It has been completely purged from all workflow definitions.
>
> **Distinction from `artifact-metadata`:** `artifact-metadata` is not a configurable workflow permission key in GitHub Actions. Actions artifact APIs operate over internal runner tokens; no `artifact-metadata` permission exists or is required.
>
> **Validation Notice:** Permissions syntax conforms to GitHub Actions workflow schema documentation. No live YAML validator was executed against a live GitHub Actions runner since no live workflow file is deployed on disk yet (documentation-only scope).

### 6.2 Verified Action Release Metadata & Dependencies

All actions are pinned to immutable 40-character commit SHAs. Metadata and dependencies have been verified against upstream repositories:

| Action | Tag | Pinned 40-Character Commit SHA | Runtime Type | Nested Dependencies | Minimum Runner | Upstream Status & Version Retention Justification |
|---|---|---|---|---|---|---|
| `actions/checkout` | `v7.0.1` | `3d3c42e5aac5ba805825da76410c181273ba90b1` | `runs.using: 'node24'` | None (native Node 24) | `>= v2.327.1` | **Upgraded to v7.0.1:** Runs natively on Node 24 (`runs.using: 'node24'`) requiring Actions Runner `>= v2.327.1`. Official documentation confirms that storing credentials in `$RUNNER_TEMP` increases security against repository artifact leaks while configuring the Git credential helper so that native Git CLI operations (`git push origin main`) succeed automatically with default `persist-credentials: true` and `permissions: contents: write`, superseding prior claims regarding `.git/config` breakage. Also enforces default security blocking untrusted fork pull request checkouts under elevated triggers. |
| `actions/setup-node` | `v7.1.0` | `949feb2413d6458794dcd2491c4babbbce0c15c1` | `runs.using: 'node24'` | None (native Node 24) | `>= v2.327.1` | Configures Node.js 24.x environment for processor and finalization scripts. |
| `actions/upload-pages-artifact` | `v5.0.0` | `fc324d3547104276b827a68afc52ff2a11cc49c9` | `runs.using: 'composite'` | GNU/BSD tar + `actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1` | `>= v2.327.1` | **Dependency Correction:** Internally delegates to `actions/upload-artifact@v7.0.1` (NOT `v4`). Packages `site/` into `artifact.tar` under `$RUNNER_TEMP`. |
| `actions/deploy-pages` | `v5.0.1` | `368f82528645a54fb793d4d04e342629a3f51346` | `runs.using: 'node24'` | None (native Node 24) | `>= v2.327.1` | Native Node 24 deployment to GitHub Pages via OpenID Connect (OIDC). |

### 6.3 Canonical Catalog Digest Input Contract (`computeCatalogDigest`)

The catalog digest provides deterministic cryptographic proof of the public store contents. To prevent scope drift and circular dependencies:

```javascript
/**
 * Computes canonical SHA-256 digest over authoritative catalog JSON files.
 * @param {string} catalogDataRoot - Root directory containing catalog data ('data' or 'site/data')
 * @returns {string} Deterministic digest formatted as 'sha256:v1:{hex}'
 */
export function computeCatalogDigest(catalogDataRoot) {
  // Canonical file inclusion list (relative to catalogDataRoot):
  // 1. catalog.json
  // 2. categories.json
  // 3. products/*.json (sorted lexicographically by filename)
  //
  // Explicitly EXCLUDED files:
  // - deployment-manifest.json (prevents circular hash dependency)
  // - HTML pages, CSS, JS, images, temporary or metadata files
}
```

- **Normal Compilation:** Reconciler invokes `computeCatalogDigest('data')`.
- **Case B Staging:** Reconciler extracts files to `site/` and invokes `computeCatalogDigest('site/data')`.
- Because both calls evaluate the exact same relative file paths and apply identical JSON canonicalization, `stagedDigest === catalogDigest` is strictly guaranteed without whole-site hash collisions.

### 6.4 Comprehensive Reconciliation & Fan-Out Query Catalog

The batch worker and reconciler execute deterministic Firestore queries backed by composite indexes:

1. **Pending Deployment Generations:**
   Discovers active compilation generations requiring build, staging, or Pages dispatch:
   ```javascript
   db.collection('system_deployments')
     .where('status', 'in', ['pending', 'catalog_committed', 'deploying'])
     .orderBy('generationSequence', 'asc');
   ```

2. **Published Generations with Incomplete Fan-Out:**
   Discovers published generations where Tier 1 global publication committed, but Tier 2 submission status fan-out remains incomplete:
   ```javascript
   db.collection('system_deployments')
     .where('status', '==', 'published')
     .where('submissionFanoutStatus', 'in', ['pending', 'in_progress', 'partial', 'failed'])
     .orderBy('generationSequence', 'asc');
   ```

3. **Pending Fan-Out Batches for a Generation (Bounded Cursor Pagination):**
   Queries uncompleted bounded batches within a specific generation using explicit page size limits:
   ```javascript
   db.collection('system_deployments')
     .doc(generationId)
     .collection('fanout_batches')
     .where('status', 'in', ['pending', 'processing'])
     .orderBy('batchIndex', 'asc')
     .limit(10);
   ```

4. **Retryable Fan-Out Batches with Lease Expiry:**
   Queries failed or expired batches where lease has expired and attempts have not exceeded 5:
   ```javascript
   db.collection('system_deployments')
     .doc(generationId)
     .collection('fanout_batches')
     .where('status', 'in', ['pending', 'processing'])
     .where('fanoutAttempts', '<', 5)
     .orderBy('batchIndex', 'asc')
     .limit(10);
   ```

5. **System-Wide Manual Review Batches:**
   Enables administrators to discover batches requiring manual intervention across all generations:
   ```javascript
   db.collectionGroup('fanout_batches')
     .where('status', '==', 'manual_review')
     .orderBy('createdAt', 'desc');
   ```

6. **System-Wide Manual Review Generations:**
   Discovers generations flagged for administrator audit:
   ```javascript
   db.collection('system_deployments')
     .where('submissionFanoutStatus', '==', 'manual_review')
     .orderBy('createdAt', 'desc');
   ```

7. **Completed Terminal Invariant:**
   Generations with `status == 'finalized'` and `submissionFanoutStatus == 'completed'` never match any active processing queries, guaranteeing they never re-enter worker execution loops.

---

## 7. Step 6: Post-Deployment Verification Checklist

Execute these checks after initial deployment:
1. **Storefront Availability:** Visit `https://rohitpvt.github.io/GitStore/` and verify that the obsidian dark UI loads without console errors.
2. **Static Catalog Access:** Confirm that `https://rohitpvt.github.io/GitStore/data/catalog.json` returns HTTP `200 OK`.
3. **Authentication Verification:** Click **[ CONNECT GITHUB ]** and verify that the GitHub OAuth consent screen opens and returns an authenticated user state.
4. **Submission Ingestion Test:** Submit a test open-source repository and verify that a document appears in Firestore `/submissions` with `status: 'queued'`. Verify that the client is prevented by security rules from setting any worker-managed fields (`validationResults`, `ownershipProof`, `processingLease`, `adminNotes`, `reviewedBy`, `reviewedAt`, `publishedProductId`).
5. **Batch Worker Execution:** Manually trigger `GitStore Batch Processor` under the GitHub Actions tab and confirm the submission transitions to `needs_review` or `approved`.
6. **Step-Up Moderation Execution:** Enqueue an approval from `admin.html` with interactive step-up reauthentication. Verify that:
   - The client computes `actionHash` (data-integrity checksum), attaches an action-specific validated payload, and generates single-use `actionNonce`.
   - The worker atomically claims the moderation request AND inserts `/consumed_stepup_nonces/{nonceHash}` using `requestedByUid`.
   - Any replay of the same nonce is rejected immediately with `NONCE_ALREADY_CONSUMED`.
   - The worker validates `targetDocumentPath` against canonical action mappings and validates the action-specific payload.
   - The worker validates GitHub collaborator rights on `Rohitpvt/GitStore` (`role_name === 'admin' || role_name === 'maintain'`).
   - The durable catalog revision transaction executes atomically (pre-reading `catalog_state`, `deployment_state`, and generation doc) and aborts if `capturedRevision < currentPublishedRevision`.
   - The static catalog compiles, stages authoritatively to `site/` with `stagedDigest` verification, commits, uploads Pages artifact via `upload-pages-artifact@v5.0.0`, passes the pre-deploy ordering guard, and deploys via `actions/deploy-pages@v5.0.1`.
