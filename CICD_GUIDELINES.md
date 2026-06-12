# CI/CD Guidelines

_Applies to all services in this organisation. Extends [DEVELOPMENT_GUIDELINES.md](./DEVELOPMENT_GUIDELINES.md) §10._  
_Effective: 2026-05-31._

---

## 1. Overview

All CI/CD runs on **GitHub Actions**. Every service has a single workflow file at `.github/workflows/ci.yml`. There is no grace period — a new service ships its pipeline on day one.

Two deployment environments exist:

| Environment | Branch | Trigger |
|---|---|---|
| `sandbox` | `dev` | Automatic on push/merge |
| `production` | `master` | Automatic on PR merge |

There is no manual promotion step. Code reaches production only through a reviewed and approved PR into `master`.

> This branch-based deployment model applies to **application service** repos. The DevOps repo (`devops`) and the documentation repo (`documentation`) are trunk-based and have **no `dev` branch** — see [DEVELOPMENT_GUIDELINES.md §11](./DEVELOPMENT_GUIDELINES.md). The DevOps repo's only CI deploy action is `terraform apply` on merge to `master` (§6); the documentation repo is not deployed.

---

## 2. Secrets & Environment Variables

### Storage

All secrets are stored as **GitHub organisation-level secrets** (Settings → Secrets and variables → Actions → Organisation secrets). Services may override at the repository level only when a value genuinely differs per repo.

Never store secrets as repository variables (those are not encrypted). Never hard-code secrets in workflow YAML, Dockerfiles, or application code.

### Required secrets (org-level)

| Secret name | Used for |
|---|---|
| `AWS_ACCESS_KEY_ID` | ECR push, EKS deploy |
| `AWS_SECRET_ACCESS_KEY` | ECR push, EKS deploy |
| `AWS_REGION` | ECR and EKS region |
| `ECR_REGISTRY` | Full ECR registry URL |
| `EKS_CLUSTER_NAME` | `kubectl` context |
| `SLACK_WEBHOOK_URL` | Deployment and rollback notifications |
| `GRAFANA_LOGS_API_KEY` | Log shipping (injected into pods, not used by CI directly) |
| `GRAFANA_API_KEY` | Metrics (injected into pods, not used by CI directly) |

### How secrets flow to pods

1. Secrets are stored in **Kubernetes Secrets** (managed by Terraform, sourced from AWS Secrets Manager).
2. The Kubernetes `Deployment` manifest references them via `envFrom` — see [DEVELOPMENT_GUIDELINES.md §8](./DEVELOPMENT_GUIDELINES.md).
3. CI never writes pod secrets directly; it only deploys manifests that reference pre-existing Kubernetes Secrets.
4. Local development uses `.env` files (gitignored). `.env.example` documents every variable.

---

## 3. Pipeline Stages

### 3.1 Pull Request pipeline

Runs on every PR targeting any branch. All steps must pass before merge is allowed.

```yaml
# .github/workflows/ci.yml (skeleton)
name: CI

on:
  pull_request:

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm

      - run: npm ci

      - run: npm run typecheck          # tsc --noEmit

      - run: npm test                   # E2E + unit (vitest)

      - run: npm run test:coverage      # Fails if API route coverage < 100%

      - run: npm audit --audit-level=high   # Blocks on high/critical

      - name: Build Docker image
        run: docker build -t ${{ env.SERVICE_NAME }}:pr-${{ github.sha }} .
        # Must succeed cleanly — no warnings treated as errors, but build failure blocks merge

      - name: Terraform plan (if infra changed)
        if: contains(steps.changed-files.outputs.all, 'infra/')
        # ... see §6 for Terraform steps
```

Steps run sequentially. If any step fails, subsequent steps are skipped and the PR is blocked.

### 3.2 Sandbox deploy (merge to `dev`)

```yaml
# Triggered by: push to dev (after PR merge)
jobs:
  deploy-sandbox:
    environment: sandbox
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Build and push Docker image to ECR
        # Tag: <service>:<git-sha>
        run: |
          docker build -t $ECR_REGISTRY/$SERVICE_NAME:$GITHUB_SHA .
          docker push $ECR_REGISTRY/$SERVICE_NAME:$GITHUB_SHA

      - name: Update Kubernetes Deployment
        # kubectl set image or helm upgrade --set image.tag=$GITHUB_SHA
        run: |
          kubectl set image deployment/$SERVICE_NAME \
            $SERVICE_NAME=$ECR_REGISTRY/$SERVICE_NAME:$GITHUB_SHA \
            -n $SERVICE_NAME

      - name: Wait for rollout (timeout: 10 min)
        run: kubectl rollout status deployment/$SERVICE_NAME -n $SERVICE_NAME --timeout=600s

      - name: Smoke test
        run: curl --fail https://$SANDBOX_HOST/healthz

      - name: Run E2E suite against sandbox
        run: npm run test:e2e
        env:
          BASE_URL: https://$SANDBOX_HOST
```

### 3.3 Production deploy (merge to `master`)

```yaml
# Triggered by: push to master (after PR merge)
jobs:
  deploy-production:
    environment: production          # Requires environment protection rules (see §7)
    steps:
      - name: Re-tag SHA image with semver
        # The semver tag is read from the git tag created by the release author
        run: |
          docker pull $ECR_REGISTRY/$SERVICE_NAME:$GITHUB_SHA
          docker tag  $ECR_REGISTRY/$SERVICE_NAME:$GITHUB_SHA \
                      $ECR_REGISTRY/$SERVICE_NAME:$SEMVER_TAG
          docker push $ECR_REGISTRY/$SERVICE_NAME:$SEMVER_TAG

      - name: Terraform apply (non-destructive changes auto-approved)
        # Destructive changes require a manual approval step — see §6

      - name: Update Kubernetes Deployment (production)
        run: |
          kubectl set image deployment/$SERVICE_NAME \
            $SERVICE_NAME=$ECR_REGISTRY/$SERVICE_NAME:$SEMVER_TAG \
            -n $SERVICE_NAME

      - name: Wait for rollout (timeout: 10 min)
        run: kubectl rollout status deployment/$SERVICE_NAME -n $SERVICE_NAME --timeout=600s

      - name: Smoke test
        run: curl --fail https://$PRODUCTION_HOST/healthz

      - name: Notify Slack on success
        uses: slackapi/slack-github-action@v1
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
          payload: |
            {
              "text": "✅ *${{ env.SERVICE_NAME }}* `${{ env.SEMVER_TAG }}` deployed to production by ${{ github.actor }}. <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View run>"
            }
```

---

## 4. ECR & Image Tagging

### Tag conventions

| Tag format | When applied | Purpose |
|---|---|---|
| `<git-sha>` (40-char) | Every build on `dev` and `master` | Precise, immutable reference |
| `v<major>.<minor>.<patch>` | After production smoke test passes | Semver release tag |
| `sandbox-latest` | After sandbox smoke test passes | Convenience for local debugging only |

Never use the bare `latest` tag in Kubernetes manifests. Always pin to a SHA or semver tag.

### Image lifecycle & cleanup

- Untagged images older than **14 days** are automatically expired via an ECR lifecycle policy (defined in Terraform).
- SHA-tagged images are retained for **90 days**.
- Semver-tagged images are retained indefinitely until manually deleted.
- Lifecycle policies are defined in `infra/modules/ecr/` and must not be modified without a team review.

---

## 5. Rollback

Rollback is **manual**. There is no automatic rollback triggered by health checks.

### When to roll back

Roll back if:
- The smoke test fails post-deploy and the issue cannot be patched immediately.
- A rollout times out (kubectl rollout status returns non-zero after 10 min).
- Production alerting fires within 30 minutes of a deploy and the deploy is the probable cause.

### How to roll back

**Option A — re-deploy via CI (preferred):**

1. Find the previous image tag (SHA or semver) in the ECR console or the previous run's CI log.
2. Trigger the `rollback` workflow manually via `workflow_dispatch`, passing the target image tag:
   ```yaml
   # .github/workflows/rollback.yml (skeleton)
   on:
     workflow_dispatch:
       inputs:
         image_tag:
           description: Image tag to roll back to (SHA or semver)
           required: true
         environment:
           description: sandbox or production
           required: true
   ```
3. The rollback workflow runs `kubectl set image` with the supplied tag, waits for rollout, and smoke-tests.
4. Post a message to `#deployments` explaining what was rolled back and why.

**Option B — kubectl directly (break-glass only):**

```bash
kubectl set image deployment/<service> <service>=<ecr-registry>/<service>:<previous-tag> -n <service>
kubectl rollout status deployment/<service> -n <service> --timeout=600s
```

Use Option B only when CI is unavailable. Document the manual action in `#deployments` immediately.

### Rollback notification

Whether rolled back via CI or manually, post to `#deployments`:

- Service name and affected environment
- Tag rolled back to
- Reason
- GitHub Actions run link (if applicable)

---

## 6. Terraform in CI

### PR pipeline

`terraform plan` runs automatically on any PR that touches `infra/`. The plan output is posted as a PR comment. The PR cannot merge if `terraform plan` exits non-zero.

```yaml
- name: Terraform plan
  if: contains(steps.changed-files.outputs.all, 'infra/')
  working-directory: infra/envs/${{ env.TARGET_ENV }}
  run: |
    terraform init -backend-config=backend.hcl
    terraform plan -out=tfplan
  env:
    TF_VAR_image_tag: ${{ github.sha }}
```

### Apply rules

| Change type | Apply method |
|---|---|
| Non-destructive (add resource, update config) | Auto-applied by CI after merge |
| Destructive (delete resource, replace, force-new) | Requires a manual approval step in the GitHub Actions environment |

A change is **destructive** if the plan output contains `destroy`, `replaced`, or `forces replacement`. CI parses the plan and gates accordingly.

### Nightly drift detection

A scheduled workflow runs `terraform plan` against both environments nightly. If the plan is non-empty, an alert is posted to the engineering Slack channel.

```yaml
on:
  schedule:
    - cron: '0 2 * * *'   # 02:00 UTC daily
```

---

## 7. Branch Protection Rules

Configure these in GitHub → Settings → Branches. Application service repos protect both `master` and `dev`; the DevOps and documentation repos protect `master` only — they have no `dev` branch (see [DEVELOPMENT_GUIDELINES.md §11](./DEVELOPMENT_GUIDELINES.md)).

### `master`

| Setting | Value |
|---|---|
| Require a pull request before merging | ✅ Enabled |
| Required approvals | 1 |
| Dismiss stale reviews on new commits | ✅ Enabled |
| Require review from code owners | ✅ Enabled (if CODEOWNERS exists) |
| Require status checks to pass | ✅ Enabled |
| Required checks | `ci / ci` (all PR pipeline jobs) |
| Require branches to be up to date before merging | ✅ Enabled |
| Block force pushes | ✅ Enabled |
| Allow deletions | ❌ Disabled |

### `dev` (application service repos only)

| Setting | Value |
|---|---|
| Require a pull request before merging | ✅ Enabled |
| Required approvals | 1 |
| Require status checks to pass | ✅ Enabled |
| Required checks | `ci / ci` |
| Block force pushes | ✅ Enabled |

### `master` on the DevOps & documentation repos

These repos are trunk-based with **no `dev` branch**. Protect `master` as above (PR required, force-push blocked, deletions disabled), with these differences:

| Setting | DevOps (`devops`) | Documentation |
|---|---|---|
| Required approvals | 1 | 2 for `*_GUIDELINES.md`, otherwise 1 |
| Required status checks | `terraform fmt -check`, `terraform validate`, `terraform plan`, Conftest | markdown lint / link check (if configured) |
| Block force pushes | ✅ Enabled | ✅ Enabled |
| Allow deletions | ❌ Disabled | ❌ Disabled |

Neither repo has a `dev` branch or a `sandbox` GitHub Environment. The DevOps repo keeps a `production` environment solely to gate `terraform apply` for destructive changes (§6).

### GitHub Environments

Configure two GitHub Environments (Settings → Environments):

- **`sandbox`** — no protection rules; auto-deploys.
- **`production`** — required reviewers: at least one team member must approve the deploy job before it runs. This is the human gate before production changes land.

---

## 8. Dependabot / Automated Dependency PRs

Enable Dependabot for npm and GitHub Actions in `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5

  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

Dependabot PRs go through the full PR pipeline like any other PR. They must pass all CI checks before a human approves and merges. Do not auto-merge Dependabot PRs — a human reviews the changelog and approves manually.

---

## 9. Multi-Service Path Filtering

If a repository contains more than one service (or both application code and `infra/`), use path filters to avoid running irrelevant jobs:

```yaml
on:
  pull_request:
    paths:
      - 'src/**'
      - 'package.json'
      - 'Dockerfile'
      - 'k8s/**'
```

For monorepos, each service folder has its own workflow file. The workflow is triggered only when files under that service's directory change.

---

## 10. Deployment Notification Format

All deploy and rollback events must be posted to `#deployments` in Slack with the following fields:

| Field | Example |
|---|---|
| Service | `scraper` |
| Version | `v1.4.2` / `abc1234` |
| Environment | `production` |
| Deployer | `@alice` |
| Status | `✅ deployed` / `⚠️ rolled back` |
| Run link | Link to the GitHub Actions run |

---

_Last updated: 2026-06-11. To propose a change, open a PR against this file and request review from at least two team members._
