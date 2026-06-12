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

### 5.1 Logging with Grafana Loki

#### Log format

Every service writes structured JSON to stdout — one JSON object per line:

```json
{
  "time": "2026-05-30T14:22:01.123Z",
  "level": "info",
  "message": "Subscription created",
  "subscriptionId": "sub_abc123",
  "userId": "usr_xyz789",
  "durationMs": 42
}
```

- `time`: ISO-8601 with milliseconds.
- `level`: one of `debug` | `info` | `warn` | `error`.
- `message`: a static human-readable string — never interpolate variable data into it. Put variable data in additional fields.
- Additional fields: camelCase, no nesting beyond one level deep.

#### Log levels

- `info` — normal milestones: service started, request received, request complete.
- `warn` — expected failure conditions: validation errors, known business errors.
- `error` — anything that caused an unhandled failure; include `error` (message only) and full request context.
- `debug` — high-frequency detail (sub-requests, cache hits); **off by default in production**.

#### What to log

| Event | Level | Required context fields |
|---|---|---|
| Service started | `info` | `port`, `nodeEnv` |
| Request received | `info` | `method`, `path`, `requestId` |
| Request completed | `info` | `method`, `path`, `status`, `durationMs`, `requestId` |
| Validation error | `warn` | `endpoint`, `field`, `requestId` |
| Expected business error | `warn` | `errorCode`, `requestId` |
| Unexpected error | `error` | `error` (message only, no stack), `endpoint`, `requestId` |
| Outbound HTTP call | `debug` | `targetService`, `method`, `url` (no query params with secrets), `status`, `durationMs` |

#### What never to log

- Passwords, tokens, API keys, or any credential.
- Full request or response bodies.
- PII (email, name, phone, address) — log IDs or hashes instead.
- Full stack traces in any field — include only `error.message`. Full stacks belong in your local debugger.

#### Grafana Loki environment variables

| Variable | Description |
|---|---|
| `GRAFANA_LOGS_URL` | Loki push endpoint from the stack's **Loki Connectivity** card (e.g. `https://logs-prod-042.grafana.net/loki/api/v1/push`). |
| `GRAFANA_LOGS_API_KEY` | `<lokiInstanceID>:<token>` pair (the Loki instance ID as username, an access-policy token with `logs:write` as password). The endpoint uses **HTTP Basic auth**, so services send `Authorization: Basic base64(GRAFANA_LOGS_API_KEY)` — _not_ `Bearer`. |

The Loki `service` stream label is taken from `GRAFANA_SERVICE_NAME` (the same canonical service identifier already used for metrics — see 5.2). Every new service must set a distinct value.

#### Shipping to Loki

Batch logs and POST to the Loki push API every **5 seconds**:

```
POST <GRAFANA_LOGS_URL>
Authorization: Basic base64(<GRAFANA_LOGS_API_KEY>)
Content-Type: application/json
```

The body is Loki JSON, grouping entries into one stream per log level. Each value is a `[<unix_ns>, "<json log line>"]` pair:

```json
{
  "streams": [
    {
      "stream": {
        "service": "<GRAFANA_SERVICE_NAME>",
        "level": "info",
        "env": "<NODE_ENV>"
      },
      "values": [
        ["1717027200123000000", "{...json log line...}"]
      ]
    }
  ]
}
```

Rules:
- If `GRAFANA_LOGS_URL` is absent, write to stdout only — never throw.
- Group entries by `level` into separate streams; timestamps are Unix nanoseconds (string).
- Maximum batch: 500 entries or 1 MB, whichever comes first. Flush early when the batch limit is reached.
- On failure (network error or non-2xx): retry once after 2 s, then drop the batch and emit a single `warn` to stdout. Never retry indefinitely or block the event loop.
- Never log the API key or any secret in failure messages.

#### Querying logs in Grafana

Use LogQL in the Grafana Explore view (Loki datasource):

```logql
# All errors for a service
{service="payments-api", level="error"}

# Slow requests (> 1 s)
{service="payments-api"} | json | durationMs > 1000

# All events for a specific request
{service="payments-api"} | json | requestId = "req_abc123"
```

#### Loki alerts

Define Loki/LogQL alert rules provisioned in each service's `grafana/` alerts file (version-controlled). Required alerts:

- Any `error`-level log from a production service → route to `#oncall` Slack immediately.

  ```logql
  count_over_time({service="payments-api", level="error", env="production"}[5m]) > 0
  ```

- More than 10 `warn`-level logs within 5 minutes from the same service → route to `#engineering` Slack.

  ```logql
  count_over_time({service="payments-api", level="warn"}[5m]) > 10
  ```

---

### 5.2 Metrics with Grafana

**Platform:** Grafana Cloud. Metrics are pushed using the InfluxDB line protocol over HTTPS.

#### Grafana environment variables

| Variable | Description |
|---|---|
| `GRAFANA_METRICS_URL` | InfluxDB write URL from the stack's **InfluxDB Connectivity** card, suffixed with `/api/v1/push/influx/write`. Note this is served on the **Prometheus host**, not a separate `influx-prod-xx` host (e.g. `https://prometheus-prod-66-prod-us-east-3.grafana.net/api/v1/push/influx/write`). |
| `GRAFANA_API_KEY` | `<instanceID>:<token>` pair (the Prometheus instance ID as username, an access-policy token with `metrics:write` as password). The endpoint uses **HTTP Basic auth**, so services send `Authorization: Basic base64(GRAFANA_API_KEY)` — _not_ `Bearer`. |
| `GRAFANA_SERVICE_NAME` | Canonical service identifier used as the metric prefix and in dashboard labels (e.g. `payments-api`) |

#### Emitting metrics

Push every **10 seconds** using the InfluxDB line protocol:

```
<measurement>,<tag_key>=<tag_value> <field_key>=<field_value> <unix_timestamp_ns>
```

Example:

```
payments-api_request,method=POST,endpoint=/api/subscriptions,status=200 count=1 1717027200000000000
```

Rules:
- **Measurement name:** `<GRAFANA_SERVICE_NAME>_<operation>`.
- **Tags:** low-cardinality, normalised values only. `endpoint` must be a route pattern (`/api/users/:id`), never a raw request path.
- **Fields:** numeric values only (`count`, `duration_ms`, `error_count`). Never put strings in field values.
- Batch all pending measurements into a single HTTP `POST` per flush. On failure, log a `warn` and discard — do not queue indefinitely.
- Gate all pushes behind a `GRAFANA_METRICS_URL` presence check so local development produces no traffic.

#### Required metrics per service

Every service must emit, at minimum:

| Measurement | Fields | Tags |
|---|---|---|
| `<service>_request` | `count=1` | `method`, `endpoint` (route pattern), `status` (HTTP code) |
| `<service>_request_duration` | `duration_ms` | `method`, `endpoint`, `status` |
| `<service>_error` | `count=1` | `endpoint`, `error_type` |

For services with background workers or queue processors, also emit:

| Measurement | Fields | Tags |
|---|---|---|
| `<service>_job` | `count=1`, `duration_ms` | `job_name`, `status` (`success`/`failure`) |

#### Dashboards

- Every service ships a Grafana dashboard definition as JSON in `grafana/dashboard.json`.
- Required panels: request rate, error rate, p50/p95/p99 latency, and any service-specific SLIs.
- Use the `$service` template variable so dashboards can be filtered by instance.
- Import the dashboard to both sandbox and production Grafana orgs during service onboarding.

#### Alerting

Define alert rules in `grafana/alerts.yaml` using Grafana's provisioning format. Required alerts:

- Error rate > 5% over a 5-minute window → severity `warning`.
- Error rate > 20% over a 2-minute window → severity `critical`.
- p99 latency > 2 000 ms over a 5-minute window → severity `warning`.
- No metrics received for > 2 minutes → severity `critical` (dead service detection).

All critical alerts must route to the `#oncall` Slack channel via the shared Grafana notification policy.

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
  component/        # React component tests (FE services only)
```

### End-to-end tests

- Spin up `createApp()` on a random port in `beforeAll`, tear down in `afterAll`.
- Make real HTTP requests against that instance — no mocking of internal modules.
- Assert on HTTP status codes, response shape, and observable side effects (database state, emitted logs, metrics).
- Cover: the golden path for every feature, all documented error codes, and any flow that touches auth or payments.
- For every E2E test that makes a request, capture `stdout` from the test server and assert: (a) a structured log line with the correct `level` was emitted, (b) no secret or PII appears in the output.

### Unit tests

- Test exported pure functions with representative inputs, edge cases, and error paths.
- Any regex, string-parsing, or data-transformation logic gets its own unit test file — these are the cases where exhaustive variation is practical and valuable.
- Input validation schemas and functions must have unit tests covering valid inputs, missing required fields, and invalid types.
- Error-class-to-HTTP-status-code mapping must be unit-tested — verify that each known error class resolves to the correct status code.
- Do not unit-test framework glue or trivial wrappers.

### Frontend component tests

Frontend services add a `src/tests/component/` layer using Vitest + `@testing-library/react`:

- Test any component that owns non-trivial state, conditional rendering, or user-interaction logic.
- Test custom hooks in isolation using `renderHook`.
- Do not test presentational leaf components that have no logic.
- Run `axe` (via `vitest-axe` or `@axe-core/react`) on every component test. A failing accessibility audit is a failing test — treat it as a blocking error, not a warning.

### Contract tests

Any type exported from `src/shared/types.ts` that crosses a service boundary (i.e. is consumed by another service's HTTP client) is a contract. Breaking contract changes require:

1. An E2E test in the consuming service that exercises the changed shape before the PR is merged in the producing service.
2. A comment in the PR referencing the consuming service and the test added.

Where multiple services share a high-churn API, use consumer-driven contract tests (e.g. [Pact](https://pact.io)) and run them in CI before the provider can merge.

### Database migration tests

Before every PR that adds or modifies a migration file in `db/migrations/`:

1. Spin up an ephemeral PostgreSQL container in CI.
2. Run all migrations from scratch (`npm run db:migrate`).
3. Run migrations again against the current schema to verify idempotency where applicable.
4. Assert the resulting schema matches expected table/column definitions (`npm run db:verify`).

Migration tests run as a separate CI job and must pass before the deploy step is reached.

### Test hygiene

- Tests must be deterministic. No `Date.now()` or `Math.random()` without injection.
- Each test is independent — no shared mutable state between test cases.
- Avoid snapshot tests for anything that changes frequently; they become maintenance noise.

### CI coverage gate

Run `npm run test:coverage` in CI. Block merges if either of the following drops:

- E2E coverage of documented API routes below **100%** (every route must have at least one E2E test).
- E2E coverage of documented error codes below **100%** (every `{ error: string }` response documented in the service README must have at least one test that triggers it and asserts the correct HTTP status code and body shape).

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

> **Temporary (current state):** the `sandbox` environment is not provisioned
> yet. Until it is, a push to `dev` runs **CI only** (typecheck, tests, build,
> audit) and does **not** deploy — service pipelines omit the `deploy-sandbox`
> job. Production behaviour is unchanged: merging into `master` still deploys.
> Restore the sandbox auto-deploy step in each service's `ci.yml` once the
> sandbox environment exists.

There is no manual promotion step between environments. Code reaches production only through a reviewed and approved PR into `master`.

> These environments and the branch-based deploys below apply to **application services**. The DevOps (`devops`) and documentation (`documentation`) repos do not deploy from a `dev` branch — see §11.1.

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

- `startupProbe`, `livenessProbe`, and `readinessProbe` pointing to `GET /healthz` (see **Health-Check Probes** below)
- `resources.requests` and `resources.limits` for CPU and memory
- `envFrom` referencing a `ConfigMap` for non-secret config and a `Secret` for secrets
- `terminationGracePeriodSeconds` set to at least 30 s to allow in-flight requests to drain

### Health-Check Probes

Every service exposes `GET /healthz`, returning `{ "status": "ok" }` once it is
ready to serve traffic. This single endpoint backs all three probes and the
post-deploy CI smoke-test. Use this exact probe block in every
`k8s/<service>/deployment.yaml`, changing only `<port>` to the container port:

```yaml
startupProbe:           # gates liveness/readiness until the app has booted
  httpGet:
    path: /healthz
    port: <port>
  periodSeconds: 10
  failureThreshold: 30  # allow up to ~5 min to come up before the pod is killed
livenessProbe:
  httpGet:
    path: /healthz
    port: <port>
  initialDelaySeconds: 15
  periodSeconds: 15
readinessProbe:
  httpGet:
    path: /healthz
    port: <port>
  initialDelaySeconds: 10
  periodSeconds: 10
```

Rules:

- **Always include the `startupProbe`.** It is what makes the short
  `livenessProbe.initialDelaySeconds` safe — without it, a service that is slow
  to boot (migrations, cache warm-up, cold JIT) can be killed by the liveness
  probe before it ever comes up, producing a crash-loop that looks like an app
  bug but is really a probe-timing bug.
- Do not lower `failureThreshold` on the liveness or readiness probes (the
  Kubernetes defaults of 3 apply); slow-boot tolerance belongs in the
  `startupProbe`, not in loosened liveness thresholds.
- `/healthz` must be cheap and dependency-light. It should report that the
  process is up — not run deep checks against databases or upstreams, which would
  make a downstream blip cascade into pods being restarted.

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

### Infrastructure testing

Every PR touching `infra/` must pass all three checks before the plan is shown for approval:

1. **`terraform validate`** — syntax and internal consistency.
2. **`terraform fmt --check`** — formatting; fail the PR rather than auto-fix silently.
3. **Conftest policy checks** — run [Conftest](https://conftest.dev) with the shared policy bundle in `infra/policies/`. Policies enforce: mandatory resource tags, no publicly-accessible S3 buckets, no security-group rules open to `0.0.0.0/0` on sensitive ports, and CPU/memory limits present on all EKS workloads.

For complex reusable modules in `infra/modules/`, add module-level tests using `terraform test` (OpenTofu 1.6+ / Terraform 1.7+). Place test files in `infra/modules/<module>/tests/`.

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

Branching rules depend on **what kind of repository** you are in. There are three classes, and they deliberately do **not** share the same model — an application service deploys from branches, but the infrastructure and documentation repos do not.

### 11.1 Repository classes

| Class | Repos | Long-lived branches | Deploys from a branch? |
|---|---|---|---|
| Application service | every service repo (`payments-api`, `bitewise`, `web2app-manager`, …) | `dev`, `master` | Yes — `dev` → sandbox, `master` → production |
| Infrastructure / DevOps | `devops` | `master` only | No — environments are Terraform directories, not branches |
| Documentation | `documentation` | `master` only | No — docs are not deployed |

The **`dev` branch exists only for application services.** The DevOps and documentation repos are trunk-based and must not carry a long-lived `dev` branch.

### 11.2 Application services

| Branch | Deploys to | Rules |
|---|---|---|
| `dev` | Sandbox (automatic) | Feature branches merge here first |
| `master` | Production (automatic on PR merge) | Reviewed PR only; never push directly |
| `feature/*` | — | Short-lived; merge to `dev` |
| `fix/*` | — | Short-lived; merge to `dev` |

- `master` is always production-ready. Never merge broken code.
- Promotion is `dev` → `master` via reviewed PR. There is no manual environment promotion step.

### 11.3 Infrastructure / DevOps repo (`devops`)

The DevOps repo is **trunk-based**. `master` is the only long-lived branch.

| Branch | Purpose | Rules |
|---|---|---|
| `master` | Single source of truth for all infrastructure | Reviewed PR only; never push directly |
| `feat/*`, `fix/*`, `chore/*`, `bootstrap/*` | Short-lived change branches | Branch from `master`, PR back into `master`, delete after merge |

- **Environments are directories, not branches.** Each environment is a Terraform composition under `terraform/envs/<env>` (e.g. `terraform/envs/prod`) with its own state and `terraform.tfvars`. The same code is applied to each environment by selecting the directory/workspace — never by maintaining a parallel long-lived branch per environment.
- A long-lived `dev` (or `staging`/`prod`) branch is a **forbidden anti-pattern** here: parallel environment branches drift and become impossible to reconcile. To test an infrastructure change safely, apply your change branch's plan against a non-production environment directory — do not create an environment branch.
- Every PR runs `terraform fmt -check`, `terraform validate`, Conftest policy checks, and `terraform plan` (posted to the PR). See §9.
- Merging to `master` runs `terraform apply`: non-destructive changes auto-apply; destructive changes require a manual approval gate. See [CICD_GUIDELINES.md §6](./CICD_GUIDELINES.md).

### 11.4 Documentation repo (`documentation`)

The documentation repo is **trunk-based** and is **not deployed**.

| Branch | Purpose | Rules |
|---|---|---|
| `master` | Published source of truth for all guidelines and docs | Reviewed PR only; never push directly |
| `docs/*`, `fix/*` | Short-lived change branches | Branch from `master`, PR back into `master`, delete after merge |

- No `dev` branch, no sandbox, no deploy step. A merge to `master` simply updates the docs.
- PRs require one reviewer approval. Guideline files (`*_GUIDELINES.md`) require **two** approvals — see the footer of each guideline.

### 11.5 Rules common to all repos

- PRs require at least one reviewer approval before merge (guideline docs require two).
- Commit messages: imperative mood, present tense. _"Add Loki log batching"_, not _"Added..."_.
- Squash-merge PRs. The PR title becomes the commit message.
- Never push directly to `master`. Never force-push a shared branch. Never delete `master`.
- Semver tags (`v1.2.3`) are created on `master` by the release author post-smoke-test (application services only).

### 11.6 Policy breaches

A **breach** is any action that bypasses the branching, review, or deploy rules above. Breaches are treated as incidents, not mistakes to be quietly fixed.

**What counts as a breach:**

- A direct push to `master` (any repo).
- A force-push to, or deletion of, any shared branch (`master`, or `dev` on a service repo).
- Merging a PR without the required approvals, or self-approving.
- Bypassing required status checks (e.g. admin-merging a red PR).
- Re-introducing a long-lived `dev`/`staging`/`prod` branch in the DevOps or documentation repos.
- Any manual change to cloud infrastructure outside Terraform — a console change ("if you clicked it, it doesn't exist", §9).

**Detection.** Branch protection (see [CICD_GUIDELINES.md §7](./CICD_GUIDELINES.md)) blocks most breaches outright. The remainder are caught by GitHub's audit log, Terraform nightly drift detection (§9), and the required-reviewers gate on the `production` environment.

**Required response.** When a breach is detected:

1. **Announce it** in `#engineering` immediately — name the repo, the branch, and the commit/PR. Breaches are surfaced, never hidden.
2. **Assess impact.** Did the change reach an environment? For infra, run `terraform plan` to measure drift; for services, check whether the change deployed.
3. **Restore the rule.** Revert the offending commit via a normal reviewed PR, or roll back the deploy (see [CICD_GUIDELINES.md §5](./CICD_GUIDELINES.md)). Never "fix forward" with a second out-of-policy push.
4. **Re-enable protection** if it was disabled, and confirm the setting matches §7.
5. **Write it up.** For any breach that reached production or caused infra drift, file a short blameless postmortem: what happened, why protection didn't stop it, and the control added so it can't recur.

**Repeated breaches** of the same rule signal a missing control, not a careless person — the postmortem must close with a concrete control (a new branch-protection setting, a CI check, a CODEOWNERS rule), not just a reminder.

> This section covers **process / policy breaches**. Security incidents and data breaches (credential leaks, unauthorised access, PII exposure) are a separate concern — handle those via the security incident-response process and §12, not this section.

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
- [ ] E2E test coverage for every documented error code (correct HTTP status + `{ error: string }` body)
- [ ] Component tests for all stateful React components (FE services only), with `axe` accessibility assertions
- [ ] Database migration test job in CI (ephemeral Postgres, migrate from scratch, verify schema)
- [ ] `.env.example` with all variables documented
- [ ] Structured logging with a distinct `GRAFANA_SERVICE_NAME`
- [ ] Grafana metrics with a distinct metric prefix
- [ ] Terraform/IaC entry in `infra/envs/sandbox/` and `infra/envs/production/`
- [ ] Rollback strategy documented (automatic via CI health-check gate)
- [ ] Service registered in `infra/shared/services.md` with its name, metric prefix, and log subsystem

---

## 14. Opening a New Repository

Follow these steps in order every time a new service is created. Do not skip steps — each one is a prerequisite for the next.

### Step 1 — Create the GitHub repo

```bash
gh repo create shaiboujuP/<service-name> --private --description "<short description>"
gh repo clone shaiboujuP/<service-name> ~/git/<service-name>
cd ~/git/<service-name>
```

Use a short, lowercase, hyphenated name (e.g. `landing-generator`, `auth-service`).

### Step 2 — Create and push the required branches

```bash
git checkout -b dev
git push -u origin dev
git checkout -b master
git push -u origin master
gh repo edit --default-branch master
```

Both branches must exist before branch protection rules can be applied.

> This `dev` + `master` setup is for **application service** repos only. The DevOps and documentation repos are trunk-based (§11.3–11.4) — create `master` alone and skip the `dev` branch entirely.

### Step 3 — Set branch protection rules

Go to **GitHub → repo → Settings → Branches** and add the following rules:

**For `master`:**
- ✅ Require a pull request before merging
- ✅ Require at least 1 approval
- ✅ Require status checks to pass (CI pipeline)
- ✅ Do not allow direct pushes

**For `dev`:**
- ✅ Require status checks to pass (CI pipeline)

### Step 4 — Create the folder structure

```bash
mkdir -p src/server src/client src/shared src/tests/e2e src/tests/unit
mkdir -p k8s .github/workflows db/migrations
touch Dockerfile .env.example package.json tsconfig.json vitest.config.ts
```

### Step 5 — Wire up the health check endpoint

Every service must have this route before anything else. Kubernetes liveness and readiness probes depend on it.

```ts
// src/server/index.ts
if (request.url === "/healthz") {
  sendJson(response, 200, { status: "ok" });
  return;
}
```

### Step 6 — Write the Dockerfile

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json ./
EXPOSE 4173
CMD ["node", "dist/server/index.js"]
```

### Step 7 — Write Kubernetes manifests

Create `k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <service-name>
  namespace: <service-name>
spec:
  replicas: 2
  selector:
    matchLabels:
      app: <service-name>
  template:
    metadata:
      labels:
        app: <service-name>
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: <service-name>
          image: <ecr-url>/<service-name>:<git-sha>
          ports:
            - containerPort: 4173
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 500m
              memory: 1Gi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 4173
            initialDelaySeconds: 15
            periodSeconds: 15
          startupProbe:
            httpGet:
              path: /healthz
              port: 4173
            periodSeconds: 10
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /healthz
              port: 4173
            initialDelaySeconds: 10
            periodSeconds: 10
          envFrom:
            - configMapRef:
                name: <service-name>-config
            - secretRef:
                name: <service-name>-secrets
```

### Step 8 — Write the CI/CD pipeline

Create `.github/workflows/ci.yml`:

```yaml
name: CI/CD

on:
  push:
    branches: [dev, master]
  pull_request:

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run typecheck
      - run: npm test
      - run: npm run test:coverage
      - run: npm audit --audit-level=high
      - name: Build Docker image
        run: docker build -t ${{ github.event.repository.name }}:${{ github.sha }} .

  deploy-sandbox:
    needs: ci
    if: github.ref == 'refs/heads/dev'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: Push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
          docker build -t ${{ secrets.ECR_REGISTRY }}/${{ github.event.repository.name }}:${{ github.sha }} .
          docker push ${{ secrets.ECR_REGISTRY }}/${{ github.event.repository.name }}:${{ github.sha }}
      - name: Deploy to sandbox
        run: |
          aws eks update-kubeconfig --name sandbox-cluster --region ${{ secrets.AWS_REGION }}
          kubectl set image deployment/<service-name> <service-name>=${{ secrets.ECR_REGISTRY }}/${{ github.event.repository.name }}:${{ github.sha }} -n <service-name>
          kubectl rollout status deployment/<service-name> -n <service-name> --timeout=120s
      - name: Smoke test
        run: curl -f https://<service>.sandbox.yourdomain.com/healthz

  deploy-production:
    needs: ci
    if: github.ref == 'refs/heads/master'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: Push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
          docker build -t ${{ secrets.ECR_REGISTRY }}/${{ github.event.repository.name }}:${{ github.sha }} .
          docker push ${{ secrets.ECR_REGISTRY }}/${{ github.event.repository.name }}:${{ github.sha }}
      - name: Deploy to production
        run: |
          aws eks update-kubeconfig --name production-cluster --region ${{ secrets.AWS_REGION }}
          kubectl set image deployment/<service-name> <service-name>=${{ secrets.ECR_REGISTRY }}/${{ github.event.repository.name }}:${{ github.sha }} -n <service-name>
          kubectl rollout status deployment/<service-name> -n <service-name> --timeout=180s
      - name: Smoke test
        run: curl -f https://<service>.yourdomain.com/healthz
      - name: Notify Slack
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
          -d '{"text":"✅ Deployed <service-name> `${{ github.sha }}` to production"}'
```

### Step 9 — Add GitHub Actions secrets

Go to **GitHub → repo → Settings → Secrets and variables → Actions** and add:

| Secret | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM key for CI (least-privilege, scoped to ECR push + EKS deploy) |
| `AWS_SECRET_ACCESS_KEY` | Corresponding IAM secret |
| `AWS_REGION` | e.g. `us-east-1` |
| `ECR_REGISTRY` | e.g. `123456789.dkr.ecr.us-east-1.amazonaws.com` |
| `SLACK_WEBHOOK` | Slack incoming webhook URL for `#deployments` |

### Step 10 — Register the service

Add a row to `infra/shared/services.md`:

```
| <service-name> | <metric-prefix>_ | <grafana-service-label> | <env> |
```

Then add a Terraform module instantiation in both `infra/envs/sandbox/main.tf` and `infra/envs/production/main.tf`.

### Step 11 — Complete the §13 checklist

Do not merge to `master` or deploy to production until every item in the New Service Checklist (§13) is checked off.

---

_Last updated: 2026-06-11. To propose a change, open a PR against this file and request review from at least two team members._
