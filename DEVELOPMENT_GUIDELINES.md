# Development Guidelines

_Applies to all services in this organisation, effective 2026-05-30._

---

## 1. Language, Runtime & Toolchain

**TypeScript everywhere.** All services — backend, frontend, scripts, and infra tooling — are written in TypeScript with `strict: true`. No `any` except at true external boundaries; use `unknown` and narrow explicitly.

**Runtime:** Node.js LTS (currently 22.x). Pin the exact version in `.nvmrc` and in the Docker base image.

**Module format:** `"type": "module"` in every `package.json`. Use ESM imports exclusively; no `require()`.

**Package manager:** npm with lockfiles committed. Do not mix package managers across services.

**Build:** Vite for client bundles, `tsc` for type-checking, `tsx` for running server code. The client builds to `dist/client`; the server is run directly with `tsx` in development and containerised for production.

**Target:** `ES2022` (`lib: ["DOM", "DOM.Iterable", "ES2022"]`). Do not downgrade for browser compat — use a modern CDN that handles this.

---

## 2. Project Structure

Every service follows this layout:

```
service-name/
  src/
    server/           # HTTP handler, routes, middleware
    client/           # React UI (if applicable)
    shared/           # Types and pure utilities shared by both sides
    tests/
      e2e/            # End-to-end tests (primary)
      unit/           # Unit tests for isolated logic
  dist/               # Build output — never committed
  k8s/                # Kubernetes manifests or Helm chart
    deployment.yaml
    service.yaml
    configmap.yaml
  .github/
    workflows/
      ci.yml
  Dockerfile
  .env.example        # All env vars documented; no defaults with secrets
  package.json
  tsconfig.json
  vite.config.ts      # (FE services only)
  vitest.config.ts
```

Shared domain types live in `src/shared/types.ts`. Interfaces that cross the HTTP boundary are defined there and imported by both server and client — never duplicated.

---

## 3. Backend

### HTTP Server

Use Node's built-in `http.createServer` or a lightweight framework (e.g. Hono) — no Express. Export a `createApp()` factory so the server can be tested without binding a port. The entry-point guard (`if (process.argv[1] === fileURLToPath(import.meta.url))`) keeps tests clean.

Every service exposes a `GET /healthz` endpoint that returns `200 OK` with `{ status: "ok" }`. Kubernetes liveness and readiness probes point to this endpoint.

### API Design

- REST for simple CRUD. JSON request/response bodies.
- All API routes under `/api/`. Static assets served from the same process in production.
- Return structured errors: `{ error: string }`. Never leak stack traces to the client.
- Validate all inputs at the API boundary before touching business logic.

### Error Handling

Wrap every async route handler in a try/catch. Map known error classes to HTTP status codes (400 for client errors, 500 for internal). Unknown errors go to `logger.error` with full context, then 500 to the client.

### External Calls

All outbound `fetch` calls get a timeout (use `AbortSignal.timeout(ms)`). Treat third-party APIs as unreliable — fail gracefully, log the raw status, surface a user-friendly message.

---

## 4. Frontend

**Framework:** React 19 with hooks. No class components.

**State:** React's built-in `useState`/`useReducer`/`useContext`. Reach for a library only when the state genuinely spans many unrelated components and the native primitives become painful.

**Styling:** Plain CSS with custom properties for theming. No CSS-in-JS. No global frameworks like Tailwind unless the team explicitly adopts one — file a guidelines amendment first.

**Icons:** `lucide-react`. Keep the import surface small; import individual icons, not the whole library.

**Type safety:** The client never hard-codes API request/response shapes — import from `src/shared/types.ts`. If a shape changes, TypeScript surfaces every affected callsite.

**Dev server:** Vite on port 5173, proxying `/api/*` to the server on 4173. Never make direct HTTP calls that bypass this proxy.

**Accessibility:** Every interactive element must be keyboard-reachable and have a visible focus state. All images need `alt` text. Test with a screen reader before shipping any non-trivial UI change.

---

## 5. Observability

### Logging

Every service uses structured JSON logging to stdout — one JSON object per line. The schema is:

```json
{ "time": "<ISO-8601>", "level": "info|warn|error|debug", "message": "<string>", ...context }
```

- `logger.info` for normal milestones (request received, request complete).
- `logger.error` for anything that caused a failure, with the error message in context.
- `logger.debug` for high-frequency detail (individual sub-requests, cache hits) — off by default in production.
- Never log secrets, tokens, full request bodies, or PII. Log identifiers (IDs, hashes) instead.

**Coralogix:** When `CORALOGIX_API_KEY` is set, logs are batched and shipped every 5 s. The application name and subsystem default to sensible values and are overridable via env vars. Every new service must set a distinct `CORALOGIX_APP_NAME`.

### Metrics

Metrics are pushed to Grafana Cloud via InfluxDB line protocol every 10 s when `GRAFANA_METRICS_URL` and `GRAFANA_API_KEY` are set.

Standard metrics every service must emit:

| Metric | Fields | Tags |
|---|---|---|
| `<service>_request` | `count=1` | `method`, `endpoint`, `status` |
| `<service>_<operation>` | `duration_ms` | `status`, relevant ID tag |

Use the service name as the metric prefix (e.g. `scraper_`, `generator_`). Tag cardinality matters — never use a user-supplied string as a tag value without normalising it first.

---

## 6. Testing

### Philosophy

**End-to-end tests are the primary testing strategy.** Every critical user flow must be covered by an automated E2E test. E2E tests run against a real running instance of the service (local or sandbox) and exercise the full stack: HTTP in, database writes, downstream calls, HTTP response.

Unit tests are a complement, not the default. Add a unit test only when there is clear value: complex business logic, critical edge cases, or isolated validation/parsing routines (e.g., the App Store URL parser, subscription price extractor) where exhaustive input variation is better handled at the unit level.

The test pyramid is intentionally flattened: fewer mocks, more real behaviour.

### Framework

Vitest for both E2E and unit tests. Config in `vitest.config.ts` at the root. Test files live in `src/tests/`:

```
src/tests/
  e2e/              # Full-stack flows against a running server
  unit/             # Pure-function tests
```

### End-to-end tests

- Spin up `createApp()` on a random port in `beforeAll`, tear down in `afterAll`.
- Make real HTTP requests against that instance — no mocking of internal modules.
- Assert on HTTP status codes, response shape, and observable side effects (database state, emitted logs, metrics).
- Cover: the golden path for every feature, all documented error codes, and any flow that touches auth or payments.

### Unit tests

- Test exported pure functions with representative inputs, edge cases, and error paths.
- Any regex, string-parsing, or data-transformation logic gets its own unit test file — these are the cases where exhaustive variation is practical and valuable.
- Do not unit-test framework glue or trivial wrappers.

### Test hygiene

- Tests must be deterministic. No `Date.now()` or `Math.random()` without injection.
- Each test is independent — no shared mutable state between test cases.
- Avoid snapshot tests for anything that changes frequently; they become maintenance noise.

### CI coverage gate

Run `npm run test:coverage` in CI. Block merges if E2E coverage of documented API routes drops below 100% (every route must have at least one E2E test).

---

## 7. Environments

Two environments exist:

| Name | Purpose | Branch | Data |
|---|---|---|---|
| `sandbox` | Integration, QA, E2E validation | `dev` | Synthetic / anonymised |
| `production` | Live traffic | `master` | Real |

**Deployment triggers:**

- Push or merge to `dev` → automatic deploy to **sandbox**.
- PR merged into `master` → automatic deploy to **production**.

There is no manual promotion step between environments. Code reaches production only through a reviewed and approved PR into `master`.

**Environment detection:** services read `NODE_ENV` (`development` / `sandbox` / `production`). Never branch on the literal string in business logic — use feature flags or config instead.

**Secrets:** never in code or git. Secrets are stored in Kubernetes Secrets (backed by a secrets manager) and injected as environment variables into pods. `.env.example` documents every variable with a description and safe placeholder value. Local development uses `.env` files (gitignored).

---

## 8. Infrastructure & Kubernetes

### Container Platform

Everything runs on **AWS EKS**. All services, workers, cron jobs, queues, and internal tooling are Kubernetes-managed unless there is a documented and approved reason not to be.

Each service is a separate Kubernetes `Deployment` with its own namespace. No shared pods across services.

### Cloud-agnostic by design

Architecture decisions must avoid deep dependency on AWS-specific managed services. The goal is that migrating to another cloud provider (GCP, Azure, Hetzner) requires only replacing the infrastructure layer — application code and Kubernetes manifests are untouched.

**Do not use:**
- DynamoDB (use PostgreSQL instead)
- AWS-specific message queues like SQS as a hard dependency in application code (abstract behind an interface)
- Lambda for core business logic (use a Kubernetes `Job` or `CronJob`)
- Proprietary AWS service SDKs embedded directly in application code

**Prefer:**
- Kubernetes-native patterns (`Deployment`, `Job`, `CronJob`, `HPA`)
- PostgreSQL-compatible databases (RDS PostgreSQL is acceptable for now; the schema and queries must be standard SQL)
- Redis-compatible caching and queuing
- Standard S3-compatible object storage APIs (use an abstraction, not the AWS SDK directly)
- Terraform / OpenTofu for infrastructure as code

### Kubernetes Manifests

Every service ships its own Kubernetes manifests in `k8s/`. These are the source of truth for how the service runs. Manifests are environment-agnostic by default; environment-specific values (image tag, replica count, resource limits, config) are applied via Kustomize overlays or Helm values files.

Mandatory manifest requirements:

- `livenessProbe` and `readinessProbe` pointing to `GET /healthz`
- `resources.requests` and `resources.limits` for CPU and memory
- `envFrom` referencing a `ConfigMap` for non-secret config and a `Secret` for secrets
- `terminationGracePeriodSeconds` set to at least 30 s to allow in-flight requests to drain

### Networking

- All inter-service traffic stays inside the cluster via Kubernetes `Service` DNS (`service-name.namespace.svc.cluster.local`).
- Public traffic enters through an ingress controller (NGINX Ingress or AWS Load Balancer Controller).
- Network policies restrict pod-to-pod traffic: a pod may only receive traffic from the ingress and from services explicitly listed in its `NetworkPolicy`.

### Storage & Databases

- **Relational data:** PostgreSQL. One RDS PostgreSQL cluster per environment. Use separate databases per service, not separate clusters. Schema managed by migration files in `db/migrations/`; migrations run as a Kubernetes `Job` before the new `Deployment` rolls out.
- **Cache / ephemeral key-value:** Redis (ElastiCache Redis or self-hosted on EKS). One cluster per environment.
- **Object storage:** S3-compatible storage accessed through a thin abstraction layer — never call the AWS SDK directly from service code. Bucket naming: `<org>-<service>-<env>`.
- Never auto-migrate on startup inside the main process.

### DNS & TLS

- Route 53 for DNS. Wildcard TLS certificate per environment managed by cert-manager (Let's Encrypt or ACM).
- HTTPS everywhere. HTTP redirects to HTTPS at the ingress.

---

## 9. Infrastructure as Code

**Terraform / OpenTofu** for all cloud resources. No manual console changes. If you clicked it, it doesn't exist.

### Repository layout

```
infra/
  modules/
    eks-service/      # Reusable: namespace, deployment, service, ingress, HPA
    rds/
    redis/
    s3/
    cert-manager/
  envs/
    sandbox/
      main.tf
      terraform.tfvars
    production/
      main.tf
      terraform.tfvars
  shared/
    vpc.tf
    eks.tf
    dns.tf
```

### Conventions

- One Terraform workspace per environment. State stored in S3 with a locking backend (DynamoDB or Terraform Cloud).
- All module inputs typed with `variable` blocks including descriptions.
- No hard-coded ARNs, account IDs, or region strings in modules — pass as variables.
- Run `terraform plan` in CI on every PR touching `infra/`. Apply only after a human approves the plan output.
- Tag every resource: `Project`, `Service`, `Environment`, `ManagedBy=terraform`.

### Drift detection

Run `terraform plan` nightly. Alert the engineering Slack channel if the plan is non-empty.

---

## 10. CI/CD

**Platform:** GitHub Actions. Every new service or service upgrade must have a CI/CD pipeline configured from day one — there is no grace period.

### Pull Request pipeline (every PR, any branch)

1. `npm ci`
2. `npm run typecheck`
3. `npm test` (E2E + unit)
4. `npm run test:coverage` (API route coverage gate)
5. `npm audit` (block on high/critical)
6. Docker image build (must succeed cleanly)
7. `terraform plan` if `infra/` is touched

### Merge to `dev` → Sandbox deploy

1. All PR checks pass.
2. Docker image built, tagged `<service>:<git-sha>`, pushed to ECR.
3. Kubernetes `Deployment` image tag updated via `kubectl set image` or a Helm upgrade.
4. Kubernetes rolls out; rollout status polled until complete or timed out.
5. Smoke test: `GET /healthz` on sandbox, assert `200`.
6. E2E test suite runs against sandbox.

### Merge to `master` → Production deploy

1. Re-tag the SHA image with the semver tag in ECR (tag created from `master` HEAD).
2. `terraform apply` for production (auto-approved for non-destructive changes; manual gate for destructive changes).
3. Kubernetes production `Deployment` updated; rollout monitored.
4. Smoke test: `GET /healthz` on production.
5. Deployment notification posted to the Slack `#deployments` channel (service name, version, deployer, link to run).

### Rollback

Every deploy records the previous image tag. If the health check or smoke test fails post-deploy, CI automatically rolls back by re-deploying the previous image tag and posts a rollback alert to Slack.

---

## 11. Git & Branching

| Branch | Deploys to | Rules |
|---|---|---|
| `dev` | Sandbox (automatic) | Feature branches merge here first |
| `master` | Production (automatic on PR merge) | Reviewed PR only; never push directly |
| `feature/*` | — | Short-lived; merge to `dev` |
| `fix/*` | — | Short-lived; merge to `dev` |

- `master` is always production-ready. Never merge broken code.
- PRs require one reviewer approval before merge.
- Commit messages: imperative mood, present tense. _"Add Coralogix batching"_, not _"Added..."_.
- Squash-merge PRs. The PR title becomes the commit message.
- Semver tags (`v1.2.3`) are created on `master` by the release author post-smoke-test.

---

## 12. Security

- `npm audit` in CI. Block on high/critical findings unless suppressed with a written justification comment.
- OWASP Top 10: validate and sanitise all user input. Never interpolate user data into SQL, shell commands, or HTML without escaping.
- No secrets in logs, metrics tags, or error responses sent to clients.
- Kubernetes pod IAM roles (IRSA) follow least-privilege. One role per service, defined in Terraform.
- Rotate all API keys and tokens on a 90-day schedule.
- Security review required (via `/security-review`) before merging any PR that adds new external HTTP calls, auth logic, or user-data handling.

---

## 13. New Service Checklist

Every new service must ship all of the following before its first production deployment:

- [ ] `Dockerfile` — builds cleanly, uses pinned Node LTS base image
- [ ] `GET /healthz` endpoint returning `{ status: "ok" }`
- [ ] Kubernetes manifests in `k8s/` (Deployment, Service, Ingress, ConfigMap, liveness/readiness probes)
- [ ] CI/CD pipeline (`.github/workflows/ci.yml`) covering build, test, Docker push, and environment deploy
- [ ] Sandbox deployment path (auto-deploy on push to `dev`)
- [ ] Production deployment path (auto-deploy on merge to `master`)
- [ ] E2E test coverage for every documented API route
- [ ] `.env.example` with all variables documented
- [ ] Structured logging with a distinct `CORALOGIX_APP_NAME`
- [ ] Grafana metrics with a distinct metric prefix
- [ ] Terraform/IaC entry in `infra/envs/sandbox/` and `infra/envs/production/`
- [ ] Rollback strategy documented (automatic via CI health-check gate)
- [ ] Service registered in `infra/shared/services.md` with its name, metric prefix, and log subsystem

---

_Last updated: 2026-05-30. To propose a change, open a PR against this file and request review from at least two team members._
