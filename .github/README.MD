# CI/CD Pipeline Architecture & Documentation

This directory contains the GitHub Actions automation, release lifecycle, and environment deployment configuration for the Vite Single Page Application.

---

## 1. High-Level Architecture & Flow

```mermaid
flowchart TD
    classDef branch fill:#2d3748,stroke:#4a5568,stroke-width:2px,color:#fff
    classDef workflow fill:#1a365d,stroke:#2b6cb0,stroke-width:2px,color:#fff
    classDef step fill:#234e52,stroke:#319795,stroke-width:1.5px,color:#fff
    classDef env fill:#744210,stroke:#d69e2e,stroke-width:2px,color:#fff
    classDef gate fill:#742a2a,stroke:#e53e3e,stroke-width:2px,color:#fff

    subgraph Phase1["1. PR Quality Gate"]
        DevBranch(["Feature / Bugfix Branch"]):::branch
        PR["Create Pull Request"]:::step
        CIWorkflow["Workflow: ci.yml"]:::workflow
        LintCheck["Lint & Types (ESLint, Stylelint, TSC)"]:::step
        FormatCheck["Format Check (Prettier)"]:::step
        TestCheck["Unit Tests (Vitest)"]:::step
        BuildCheck["Production Build (Vite)"]:::step
        
        DevBranch --> PR --> CIWorkflow
        CIWorkflow --> LintCheck & FormatCheck & TestCheck & BuildCheck
    end

    subgraph Phase2["2. Main Merge & Development Deployment"]
        MainBranch(["Merged to 'main'"]):::branch
        ReleaseWorkflow["Workflow: release.yml"]:::workflow
        CIOnMain["Workflow: ci.yml (Gate)"]:::workflow
        DeployDev["deploy.yml: Development"]:::workflow
        DevEnv[("Environment: development")]:::env
        ReleasePlease["Release Please Action"]:::step
        ReleasePR["Automated Release PR\n(Bumps Version & CHANGELOG.md)"]:::step

        MainBranch --> ReleaseWorkflow
        ReleaseWorkflow --> CIOnMain
        CIOnMain --> DeployDev
        DeployDev --> DevEnv
        CIOnMain --> ReleasePlease
        ReleasePlease --> ReleasePR
    end

    subgraph Phase3["3. Release & Multi-Environment Promotion"]
        MergeRelease["Merge Release PR / Push hotfix/**"]:::branch
        CreateTag["Git Tag Created (vX.Y.Z)\n& GitHub Release Published"]:::step
        DeployStaging["deploy.yml: Staging"]:::workflow
        StagingEnv[("Environment: staging")]:::env
        ApprovalGate{"Manual Review Gate\n(GitHub Environment Protection)"}:::gate
        DeployProd["deploy.yml: Production"]:::workflow
        ProdEnv[("Environment: production")]:::env

        ReleasePR -. "Developer Merges" .-> MergeRelease
        MergeRelease --> CreateTag
        CreateTag --> DeployStaging
        DeployStaging --> StagingEnv
        StagingEnv --> ApprovalGate
        ApprovalGate -- "Approved by Reviewer" --> DeployProd
        DeployProd --> ProdEnv
    end
```

---

## 2. Core Workflows Explained

### A. CI Workflow (`ci.yml`)
- **Trigger**: Runs on every `pull_request` event targeting branches or called reusable via `workflow_call`.
- **Purpose**: Fast feedback loop preventing broken code, syntax errors, or regression bugs from landing in protected branches.
- **Jobs & Steps**:
  1. Checks out repository source code.
  2. Sets up Node.js runtime according to `.nvmrc` with native `pnpm` caching.
  3. Installs dependencies using `pnpm install --frozen-lockfile`.
  4. Runs code quality tasks in parallel/sequence:
     - `pnpm lint`: Lints JavaScript, TypeScript, CSS/SCSS, and runs static type-checking.
     - `pnpm format:check`: Validates formatting against Prettier standards.
     - `pnpm test`: Runs Vitest unit tests.
     - `pnpm build`: Verifies that Vite compiles static output without errors.
- **Concurrency**: Cancels earlier queued runs on the same branch if a newer commit is pushed (`cancel-in-progress: true`).

---

### B. Release Orchestrator (`release.yml`)
- **Trigger**: Runs automatically on pushes to `main` and `hotfix/**`.
- **Purpose**: Coordinates deployments across environments and automates Semantic Versioning (SemVer).
- **Execution Lifecycle**:
  1. **`ci` Job**: Verifies test suite passes on the target branch.
  2. **`deploy-development` Job**: Immediately triggers `deploy.yml` with the latest commit SHA and deploys to the `development` environment.
  3. **`release-please` Job**:
     - Evaluates Conventional Commits (e.g., `feat:`, `fix:`, `perf:`).
     - Maintains or updates a tracking Release PR with updated versions in `package.json` and generated release notes in `CHANGELOG.md`.
     - When the Release PR is merged, it tags the commit with `vX.Y.Z` and outputs `release_created: true`.
  4. **`deploy-staging` Job**: Triggers once a release is created, deploying the tag to the `staging` environment.
  5. **`deploy-production` Job**: Waits for `staging` deployment to succeed, then pauses for manual approval via GitHub Environment protection before deploying to `production`.

---

### C. Reusable Deploy Workflow (`deploy.yml`)
- **Trigger**: Called by `release.yml` (`workflow_call`) or run manually (`workflow_dispatch`).
- **Inputs**:
  - `environment`: `development` | `staging` | `production`.
  - `ref`: Git branch, tag, or commit SHA.
- **Steps**:
  1. **Environment Validation**: Asserts required environment variables (e.g., `VITE_BACKEND_BASE_URL`) exist for the target environment.
  2. **Asset Compilation**: Builds optimized production bundles using Vite, embedding target environment name, release version, and short Git commit SHA.
  3. **Deploy Simulation (POC Mode)**:
     - In production, this runs `aws s3 sync` and invalidates AWS CloudFront caches.
     - In this playground/POC, it formats the exact AWS CLI commands, outputs a summary table to `$GITHUB_STEP_SUMMARY`, and uploads the build artifact (`site-<env>`).
  4. **Artifact Archiving**: Stores the compiled `dist/` directory as a downloadable GitHub Action artifact (7-day retention).

---

## 3. Environment Progression Matrix

| Environment | Trigger Condition | Code Reference | Approval Required? | Purpose |
| :--- | :--- | :--- | :---: | :--- |
| **development** | Push to `main` branch | Commit SHA (`HEAD`) | No | Continuous integration testing and active feature validation. |
| **staging** | Merging a Release PR | Release Tag (`vX.Y.Z`) | No | Pre-production testing, QA smoke tests, and final dress rehearsal. |
| **production** | Successful staging deploy | Release Tag (`vX.Y.Z`) | **Yes** (Required Reviewer) | Live customer-facing release. |

---

## 4. Manual Rollbacks & Redeploys

If an issue occurs in `staging` or `production`:
1. Navigate to **Actions** $\rightarrow$ **Deploy** workflow in GitHub.
2. Click **Run workflow**.
3. Select the target **Environment** (`staging` or `production`).
4. Enter the desired stable tag or commit (e.g., `v1.2.0`).
5. Click **Run workflow**. The deployment pipeline will compile and deploy that exact version.
