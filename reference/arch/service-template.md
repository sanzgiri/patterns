# Service Template

**Aliases:** Starter Project, Scaffold, Cookiecutter, Software Template
**Category:** Microservices / Developer productivity
**Sources:**
[Chris Richardson — microservices.io: Service Template](https://microservices.io/patterns/service-template.html) ·
[Backstage Software Templates documentation](https://backstage.io/docs/features/software-templates/) ·
Sam Newman, *Building Microservices* (2nd ed.), Ch on developer productivity

---

## Problem

> [!TIP]
> **ELI5.** Every new microservice needs the same boring stuff: a Dockerfile, a CI pipeline, logging setup, metrics endpoint, health checks, auth middleware, deployment manifests. If every team builds these from scratch, you get 50 slightly-different services, none done perfectly. A **service template** is a pre-built starter project: "run one command, get a fully-working microservice with all the cross-cutting concerns done correctly." Then you just add your business logic.

A microservices organization typically has dozens or hundreds of services. Each new service needs the same infrastructure scaffolding:

- A reproducible build (Dockerfile, build scripts).
- A CI pipeline (lint, test, build, scan, deploy).
- Observability setup (structured logging, metrics endpoint, tracing initialization, health checks).
- Resilience defaults (timeouts, retry, circuit breakers, graceful shutdown).
- Security setup (auth middleware, TLS, secret management).
- Deployment artifacts (Helm chart or Kustomize overlays, environment configs).
- Documentation skeleton (README, ADRs, OpenAPI spec, runbook).
- Standard testing harness (unit + integration test setup).

If every team builds this from scratch, three bad things happen:

1. **Inconsistency.** Service A uses Logback with one log format; Service B uses zap with a different format. Aggregation, search, and dashboards all break. Each team makes different decisions about retry strategies, timeout values, naming conventions, file layout.
2. **Quality drift.** Production-grade cross-cutting concerns are hard to do well. Some teams will skip them, do them wrong, or forget important ones (auth on the metrics endpoint, graceful shutdown). Quality of foundational scaffolding ends up bimodal: a few done very well, many done badly.
3. **Slow time-to-first-commit.** A new service takes weeks to set up before anyone writes business logic. Engineers burn out on yak-shaving before they can deliver value.

The **Service Template** pattern solves this by providing a single, curated starter project that any engineer can scaffold from. The template captures the organization's collective wisdom: best practices for observability, deployment, security, resilience — pre-wired, tested, and ready to ship.

## How it works

> [!TIP]
> **ELI5.** Build one template repository that has everything a new microservice should have: the right Dockerfile, the standard CI pipeline, the observability libraries already initialized, the deployment manifests. Add a script that scaffolds a new project from this template — "scaffold new-service order-svc" — and you get a fresh repository with a working microservice. Engineers just add business logic.

A service template is typically a code repository plus a scaffolding tool. The repository contains:

![Service Template contents](../diagrams/svg/service-template.svg)

- **Standard code layout** — `/src` for business logic, `/api` for handlers, `/domain` for entities, `/repo` for persistence, `/tests`, `/infra` for Terraform and Helm. Same layout in every service makes it easy for any engineer to navigate any service.
- **Observability pre-wired** — structured JSON logging with correlation IDs, Prometheus `/metrics` endpoint, OpenTelemetry tracing initialized, `/health` and `/ready` endpoints implemented.
- **Resilience defaults** — circuit breaker library wrapping outbound calls, retry-with-backoff, mandatory timeouts on every remote call, graceful shutdown on `SIGTERM`.
- **Security defaults** — JWT validation middleware, TLS certificate loading from cert-manager, secret injection from Vault or AWS SSM, rate-limit middleware, audit-log middleware.
- **Deployment artifacts** — multi-stage Dockerfile producing a small image, Helm chart with environment-specific values, CI workflow (GitHub Actions or similar) for lint → test → build → push → deploy, `.env.example` with all required variables.
- **Documentation skeleton** — `README.md` with sections to fill in, `RUNBOOK.md` for on-call, `ARCHITECTURE.md`, `ADRs/` directory with first decision pre-recorded.

The scaffolding tool ranges from a shell script to a sophisticated platform — Spring Initializr, Yeoman generators, Cookiecutter, GitHub's repository templates, or Backstage's Software Templates. The interaction is typically:

```
scaffold new-service \
  --name=order-svc \
  --owner=orders-team \
  --language=go
```

In ~30 seconds, the engineer has a fresh repository with a working microservice — it builds, runs, exposes `/health`, ships logs and metrics, and deploys to the dev cluster. The engineer immediately starts adding business logic, not infrastructure.

### Template content debates

Every organization argues about what belongs in the template:

- **Multiple language templates** vs **a single primary language**? Most orgs end up with templates per primary language (Go, Java, Python, Node), each maintained by language guilds.
- **Highly opinionated** vs **flexible**? Highly opinionated templates impose consistency at the cost of edge-case flexibility. The common compromise: opinionated by default, with extension points for genuine needs.
- **Include framework choices** vs **leave them open**? Most teams pre-select a framework (Spring Boot for Java, Gin or Echo for Go, FastAPI for Python) — the template is wedded to a specific stack.

The right answer depends on the organization's size and culture. A 50-engineer startup can pick one stack and stick with it. A 5,000-engineer enterprise needs multiple templates, often per business unit.

### Update propagation

The most-discussed weakness of service templates: **templates are scaffolded once, but improvements need to reach every service**. If you fix a security bug in the template's auth middleware, how does every existing service get the fix?

Common strategies:

- **No automatic updates.** Each team is responsible for periodically re-syncing. Realistic but slow; many services drift over time.
- **Sync tooling.** Tools like `cruft` (for Cookiecutter), `copier`, or homegrown sync scripts compare a service against the template and offer to update specific files. Effective for files that haven't been customized.
- **Update via library version bumps.** Push most cross-cutting concerns into a [Microservice Chassis](microservice-chassis.md) library that services depend on. Updates happen via dependency-version bumps — far more tractable than template re-scaffolding.
- **Automated PR generation.** Platform teams generate PRs against every service when a template change happens (Renovate, Dependabot, custom bots). Service teams review and merge.

Most mature organizations end up using **template + chassis together**: the template scaffolds the *structural* parts (layout, Dockerfile, CI workflow) that are mostly static, while the chassis library handles the *runtime* concerns (logging, metrics, tracing, circuit breakers) that need to evolve with the organization.

### Service Template vs Microservice Chassis

These two patterns are commonly confused but are complementary:

![Service Template vs Microservice Chassis](../diagrams/svg/template-vs-chassis.svg)

- A **Service Template** is a *one-time scaffold* of files that produces a new repository.
- A **Microservice Chassis** is a *runtime library/framework* dependency that the scaffolded service uses.

Most teams use both. The template scaffolds the repository structure, dependencies, and CI; the chassis provides the runtime cross-cutting concerns. Updating the template affects only newly-created services; updating the chassis affects every running service that bumps the dependency version.

### Beyond microservices: developer platforms

Modern internal developer platforms generalize the Service Template pattern across the entire developer experience. Tools like **Backstage** (Spotify, open-sourced), **Port**, and **Cortex** treat templates as first-class: not just for microservices, but for frontend apps, data pipelines, ML projects, infrastructure modules. The template becomes an opinionated *path of least resistance* — the easiest way to start anything new is also the right way.

Backstage's Software Templates, for example, can:
- Scaffold a new service repository.
- Pre-populate a CI/CD pipeline.
- Register the service in the developer portal (with owners, on-call, docs).
- Provision the cloud infrastructure (database, queue, secrets) via Terraform.
- Configure observability dashboards.

All in a few minutes, from a single web form. This is what "platform engineering" looks like in practice — and the Service Template pattern is its kernel.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Cookiecutter-style** | Local CLI tool; renders templates with variables. Simple. |
| **GitHub repo template** | Repository marked as a template; "Use this template" button creates a copy. |
| **Backstage Software Template** | Full developer-portal scaffolding with metadata registration. |
| **Spring Initializr-style** | Web UI lets the engineer pick options, downloads a configured starter. |
| **Per-language template** | One template per primary language; common in polyglot orgs. |
| **Per-architectural-style template** | Different templates for API service, worker, batch job, ML pipeline. |
| **Template + Chassis combo** | The dominant pattern in mature orgs. |
| **Live template (syncing)** | Tools (`cruft`, `copier`) keep generated projects in sync with template updates. |

## When NOT to use

- **Very few services** (1–3) — overhead of building/maintaining the template exceeds benefit.
- **Highly heterogeneous service requirements** — no two services share enough scaffolding for a template to help.
- **One-off prototypes and POCs** — strict template usage can slow exploration.
- **Without organizational buy-in** — a template no one uses is dead weight.

---

## Real-world implementations

| Tool | Approach |
|---|---|
| **Spring Initializr** | Web-based template for Spring Boot projects. The reference for "configure-and-download" templates. |
| **Cookiecutter** | Python-based local template renderer; widely used in Django and beyond. |
| **Yeoman** | JS-based generator framework; the original mainstream scaffolder. |
| **GitHub repository templates** | Built-in "Use this template" feature; minimal but widely used. |
| **Backstage Software Templates** | Spotify's open-source developer portal; full platform-grade scaffolding. |
| **Port** | Commercial developer-portal alternative. |
| **JHipster** | Opinionated full-stack scaffold for Spring + Angular/React. |
| **Nx generators** | Monorepo-aware scaffolding for JS/TS projects. |
| **`create-react-app`, `vue create`, `next-app`** | Single-framework scaffolders, same idea. |
| **`dotnet new` templates** | Microsoft's CLI-based templating system, language-/framework-agnostic. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Spotify** | Built Backstage internally; service templates underlie their developer platform. | ✅ Verified — [Backstage docs and Spotify Engineering posts](https://backstage.io/) |
| **Pivotal / VMware Tanzu customers** | Spring Initializr is the de-facto template tool for Java microservices. | ✅ Verified — Spring Initializr stats |
| **Netflix** | Internal scaffolders for service creation (described in tech blog posts). | ✅ Verified — Netflix Tech Blog |
| **Airbnb** | Service Hub and internal scaffolding for new services. | ✅ Verified — Airbnb Engineering posts |
| **Twilio** | Internal "service factory" generating microservices from templates. | ⚠ Mentioned in conference talks |
| **HashiCorp** | Uses Terraform module templates / scaffolders for infrastructure projects. | ✅ Verified — Terraform Registry templates |
| **Many enterprise platform-engineering teams (Goldman Sachs, ING, JPMorgan)** | Internal developer platforms with mandated service templates. | ✅ Verified — public talks at QCon, KubeCon, IDP Conf |

---

## Further reading

- Chris Richardson, *microservices.io* — Service Template pattern entry.
- Sam Newman, *Building Microservices* (2nd ed.) — Ch on Developer Productivity, including templates and chassis.
- Backstage Software Templates documentation — the most-cited modern implementation.
- *Team Topologies*, Skelton & Pais — discusses the role of platform teams in providing templates.
- *Platform Engineering* (Camille Fournier and others) — broader context for service templates within IDPs.
- Brendan Burns, *Designing Distributed Systems* — discusses "container patterns" that often appear in templates.
- The Backstage blog at Spotify Engineering — many concrete case studies on template adoption.

---

*Diagram sources: [`../diagrams/src/service-template.d2`](../diagrams/src/service-template.d2), [`../diagrams/src/template-vs-chassis.d2`](../diagrams/src/template-vs-chassis.d2).*
