# Externalized Configuration

**Aliases:** External Config, 12-Factor Config, Config-as-Code, Runtime Configuration, Dynamic Config
**Category:** Operations / Configuration
**Sources:**
[Microsoft Azure — External Configuration Store pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/external-configuration-store) ·
[microservices.io — Externalized Configuration](https://microservices.io/patterns/externalized-configuration.html) ·
[The Twelve-Factor App — III. Config](https://12factor.net/config) ·
[HashiCorp Consul documentation](https://developer.hashicorp.com/consul/docs) ·
[HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs) ·
[AWS AppConfig documentation](https://docs.aws.amazon.com/appconfig/) ·
[Spring Cloud Config](https://spring.io/projects/spring-cloud-config)

---

## Problem

> [!TIP]
> **ELI5.** Imagine you have to build a different version of your app for dev, staging, prod-US, and prod-EU because the database URLs, API keys, and feature settings are baked into the code. Now you can't promote the staging build to prod — you have to rebuild. If you check the prod password into git, you've leaked it forever. If you change a rate limit, you have to redeploy. **Externalized configuration** is the simple-sounding rule: don't put environment-specific values in code. Read them at runtime from the environment, files, or a config service. The same binary then runs everywhere; secrets stay out of source; you can change configuration without redeploying.

The pattern's importance is easy to underestimate because the basic idea ("read config at runtime") sounds trivial. The reality is that **how you externalize config shapes how your operations work**:

- **If config is in code**: changing it requires a redeploy. Promoting a build between envs is impossible. Secrets leak into source. Different envs use different binaries (so they're not really tested the same).
- **If config is in env vars only**: simple and 12-factor, but you can't change it without restart, and managing many env vars across many services becomes painful.
- **If config is in a central service**: changes propagate instantly across the fleet, but you've added a runtime dependency.
- **If secrets are mixed with non-secrets**: you can't audit safely; you can't rotate keys easily.

The pattern emerges from observing that **environment-specific values and code change at completely different cadences**. A new code release happens daily; a database password rotation happens quarterly; a rate-limit adjustment happens during an incident at 3am. These shouldn't be coupled.

The Twelve-Factor App's third factor ("Config: Store config in the environment") codified this for the cloud era in 2011. Microsoft's External Configuration Store pattern and microservices.io's Externalized Configuration pattern formalize the architecture. In practice, modern systems combine several mechanisms — static env vars for boot-time settings, a config service for runtime-tunable values, a secrets store for credentials.

## How it works

> [!TIP]
> **ELI5.** Build your binary or container once with NO environment values inside. At startup, read config from environment variables, config files, or call out to a config service. The binary is identical across dev, staging, and prod — only the config differs. Secrets come from a separate, locked-down secrets store. To change behavior, change config — no redeploy of code.

### The basic shape

A 12-factor application reads its configuration from the environment:

![Externalized configuration architecture](../diagrams/svg/externalized-config.svg)

The build artifact (Docker image, JAR, binary) contains no environment-specific values. At startup (and optionally during runtime), the application reads config from one or more sources:

- **Environment variables** (the 12-factor canonical store).
- **Config files** mounted at known paths (YAML, JSON, TOML, HCL).
- **CLI flags** for boot-time overrides.
- **Distributed KV stores** (Consul, etcd, ZooKeeper).
- **Cloud config services** (AWS AppConfig, Azure App Configuration, GCP Runtime Config).
- **Secrets stores** (Vault, AWS Secrets Manager, GCP Secret Manager) for sensitive values.

The result: the same artifact runs in dev (DB=localhost, log=DEBUG), staging (DB=stg-db.eu, log=INFO), prod-US, and prod-EU — each just with different config injected. Promotion between environments is just moving the artifact, never rebuilding.

### Static vs hot-reload vs dynamic

There's a spectrum of how dynamic the config can be, with corresponding complexity:

![Config spectrum](../diagrams/svg/config-spectrum.svg)

**Static config** — env vars or files read once at startup. Simple. Restart required to change anything. Predictable: a running process's behavior cannot change. Use for stable settings: DB connection strings, listen ports, cluster name.

**Hot-reload config** — files re-read on signal (SIGHUP, inotify watch). No restart needed; operator triggers reload. Use for things you might twiddle during an incident without full redeploy: log levels, rate limit values, lists of blocked IPs.

**Dynamic config service** — central service pushes config changes to subscribers in real time. Instant fleet-wide propagation; per-instance / per-region / per-user targeting; centralized audit. Adds a runtime dependency. Use for feature flags, A/B tests, kill switches, regional toggles, multi-tenant per-tenant overrides.

Most non-trivial systems use all three: static for things that never change, hot-reload for ops-tunable values, dynamic config for feature flags and runtime toggles.

### The 12-factor approach

The Twelve-Factor App's third factor is short and emphatic:

> Apps sometimes store config as constants in the code. This is a violation of twelve-factor, which requires strict separation of config from code. Config varies substantially across deploys, code does not. [...] The twelve-factor app stores config in environment variables (often shortened to env vars or env).

The reasoning:
- Env vars are language- and OS-agnostic.
- They're never accidentally checked into source.
- They're orthogonal: each deploy can have its own set with no merging needed.
- They're easy to change without code changes.

**Limitations** of pure env-var config:
- Doesn't scale well past ~50 vars (hard to manage).
- No types or validation (everything is a string).
- No structure (no nested objects).
- No runtime change without restart.
- Secrets shown in process listings (`ps`), Kubernetes pod manifests, etc.

So the modern stack typically uses env vars for **bootstrap** config (pointer to config service URL, environment name, log level), with the bulk of config pulled from a config service and secrets pulled from a secrets store.

### Config services in practice

A distributed config service provides:

- **Centralized storage**: all services' config in one place.
- **Hierarchical / scoped overrides**: global defaults, then per-env, then per-service, then per-instance overrides.
- **Real-time push**: subscribers get updates within seconds.
- **Version history and audit**: who changed what when.
- **Validation and schema**: prevent bad config from being deployed.
- **ACL**: not everyone can change every config key.
- **Multi-tenant / regional overrides**: different values per cell, per region, per customer.

Major options:

| Tool | Notes |
|---|---|
| **Consul** | HashiCorp; KV + service discovery; very common in cloud-native. |
| **etcd** | CNCF; Kubernetes's config store; ZooKeeper alternative. |
| **ZooKeeper** | Older; powers Kafka, HBase, others. |
| **AWS AppConfig** | Managed config + feature flag service. |
| **AWS Systems Manager Parameter Store** | Hierarchical KV; popular for AWS-based services. |
| **Azure App Configuration** | Managed config + feature flag service. |
| **GCP Runtime Config / Secret Manager** | GCP equivalents. |
| **Spring Cloud Config** | Java/Spring ecosystem; git-backed. |
| **Apollo Config (Ctrip)** | Open source; very popular in China. |
| **Confidant, Doppler, Infisical, Bitwarden Secrets Manager** | Secrets-focused. |

The choice depends on existing infrastructure. AWS shops use Parameter Store / AppConfig; Kubernetes shops use ConfigMaps + Consul/etcd; Java shops use Spring Cloud Config.

### Secrets — a special case

Secrets are configuration but have stricter requirements:

- **Encrypted at rest** in the store.
- **Encrypted in transit** when fetched.
- **Audited access**: every read logged.
- **Rotation**: must support periodic credential rotation.
- **Short-lived credentials**: ideally, services fetch a *just-in-time* token rather than a long-lived secret.
- **Never in env vars or files** if avoidable (process listing, container manifests).
- **Never in git** — even if the repo is private.

Modern stack:
- **HashiCorp Vault**: gold standard; dynamic secrets, lease-based, transit encryption.
- **AWS Secrets Manager** / **AWS KMS**: AWS-native.
- **GCP Secret Manager**: GCP equivalent.
- **Azure Key Vault**: Azure equivalent.
- **Kubernetes Secrets**: convenient but only base64-encoded — pair with external store for serious use.
- **SOPS / Sealed Secrets**: encrypt-in-repo approaches (config-as-code with encryption).

A common modern pattern: app starts → reads env var pointing to secrets store → authenticates (via cloud IAM role / Kubernetes service account / Vault auth) → fetches needed secrets → keeps them only in memory.

This way, even if the container image leaks, no secrets are exposed; even if the secret store is breached, individual instances had narrow access.

### Config-as-code

A trend: rather than configuring via UIs, store config in a git repo and apply it via CI/CD. Benefits:

- **Versioned**: full history of changes.
- **Reviewed**: PRs gate config changes.
- **Auditable**: who approved, when.
- **Reproducible**: re-apply to recreate environment.

Tools:
- **Terraform**: infrastructure as code; also config.
- **Helm + values.yaml**: Kubernetes config.
- **Kustomize**: K8s config overlays per env.
- **Ansible, Chef, Puppet**: legacy but still used.
- **GitOps** (Argo CD, Flux): config in git is automatically applied.

The flip side: changes go through git → CI → apply → service reads. That's much slower than "click in dashboard, change is live in 3 seconds." For incident response (kill switch flip), git-ops is too slow; pair config-as-code for stable config with a real-time config service for runtime toggles.

### Bootstrap problem

A circular dependency: the app needs config to start, but the config comes from a service. What does it need to know to find the config service?

The bootstrap config (env vars or local file) typically contains:
- Config service URL or cluster endpoint.
- Auth credentials or instance identity (cloud metadata).
- Service name and environment.
- Log level (so failures are visible before more config arrives).

Once these basics are there, the app fetches the rest. Bootstrap config should be minimal — ideally one env var per concern.

### Caching and fallback

What happens if the config service is down at startup?

- **Aggressive caching**: SDK caches the last known config to disk. App can start with stale config if service unreachable.
- **Bundled defaults**: code includes sensible defaults for every flag, used if both cache and service fail.
- **Graceful degradation**: services that *require* config to function (e.g., the DB connection string) must fail fast and clearly; services that can run with stale config should do so.
- **Retry with backoff**: don't hammer a recovering config service.

Without these, the config service becomes a single point of failure for every service that depends on it.

### Validation

A bad config push is a deploy. A bad deploy is an incident. So config changes need similar gates:

- **Schema validation**: reject configs with wrong types or missing required keys.
- **Linting**: reject suspicious changes (sudden 10× rate limit change).
- **Per-env approvals**: dev free, prod requires review.
- **Canary config rollout**: push new config to one instance / cell first, observe, then propagate.
- **Audit log**: who pushed what when.

AWS AppConfig, LaunchDarkly, and similar services include validators, canary rollouts, and rollback hooks for exactly this reason.

### Multi-tenant and per-instance overrides

Modern config services support overrides at multiple scopes:

```
global: { rate_limit: 1000 }
env=prod: { rate_limit: 5000 }
region=eu-west: { rate_limit: 3000 }
instance=server-42: { rate_limit: 100 }   # canary
tenant=acme-corp: { rate_limit: 50000 }   # big customer
```

The runtime resolves the most-specific applicable value. This is what makes config services more powerful than env vars — they support targeting and overrides that flat env vars can't.

### Trade-offs

Advantages:
- **Same artifact across envs** — promotion just moves the artifact.
- **Secrets stay out of source** — security by structure.
- **Runtime change without redeploy** — incident response, feature flags.
- **Per-env / per-region / per-tenant targeting**.
- **Audit and rollback** for config changes.
- **Aligns with 12-factor** and modern cloud-native patterns.

Disadvantages:
- **Runtime dependency** on config / secrets services.
- **Complexity** of managing many config keys, scopes, overrides.
- **Bootstrap problem** — how does the app find the config service?
- **Cache invalidation** — when config changes, how fast does the fleet see it?
- **Auditability across many services** can become hard.

### Anti-patterns

- **Hard-coded production values** in code — defeats the pattern.
- **Per-env code branches** (`if env == 'prod'`) — config-driven branching is fine; environment-driven branching in code is not.
- **Secrets in env vars** — visible in process listings, K8s manifests.
- **Secrets in git** — even private repos eventually leak.
- **No fallback** when config service is down — hard dependency on uptime.
- **Configs that change behavior dangerously** without canary rollout.
- **Stale config caches** with no expiry — services can run with old config forever.
- **Mixing static and dynamic concerns** — DB connection strings should not be in feature-flag service; feature flags should not be in env vars.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Env vars only (12-factor strict)** | Simplest; doesn't scale to large config sets. |
| **Config files in image / mounted** | Versioned config; restart required. |
| **Hot-reload files** | File change triggers re-read. |
| **Config service** | Real-time central config. |
| **Secrets store** | Specialized for credentials. |
| **GitOps / Config-as-code** | Config in git, applied by CI. |
| **[Feature Flag](feature-flag.md)** | Specialized config for releases / experiments. |
| **Service Discovery** ([page](../comm/service-registry.md)) | Specialized config for "where are services?" |
| **Stamp Router** ([page](deployment-stamps.md)) | Specialized config for tenant routing. |

## When NOT to use

- **Truly small / hobby apps** — env vars suffice.
- **Single-environment apps** — less benefit.
- **Embedded systems** with no runtime config service.
- **Without a security model** for secrets — better to ship hard-coded than do secrets badly.

---

## Real-world implementations

| Tool | Type |
|---|---|
| **HashiCorp Consul** | KV + service discovery |
| **HashiCorp Vault** | Secrets |
| **etcd** | KV; Kubernetes's store |
| **Apache ZooKeeper** | KV; older; still used (Kafka, HBase) |
| **AWS AppConfig** | Config + flags |
| **AWS SSM Parameter Store** | Hierarchical KV |
| **AWS Secrets Manager + KMS** | Secrets |
| **Azure App Configuration** | Config + flags |
| **Azure Key Vault** | Secrets |
| **GCP Runtime Config / Secret Manager** | Config + secrets |
| **Spring Cloud Config** | Java-centric |
| **Apollo Config (Ctrip)** | Popular OSS |
| **LaunchDarkly / Unleash / GrowthBook** | Flag-focused with config |
| **Doppler, Infisical, Bitwarden SM, 1Password CLI** | Secrets-focused |
| **Kubernetes ConfigMap + Secret** | K8s-native (basic) |
| **SOPS, Sealed Secrets, ExternalSecrets operator** | GitOps for secrets |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Netflix** | Archaius (open-source dynamic property library). | ✅ Verified — Netflix OSS |
| **Twitter / X** | Decider (dynamic config service); described in talks. | ✅ Verified — Twitter Engineering blog |
| **Facebook / Meta** | Configerator + Gatekeeper for config + flags at scale. | ✅ Verified — Engineering posts |
| **Google** | Server-side experiment framework; integrated with config. | ✅ Verified — Tang et al. KDD 2010 |
| **Airbnb** | Built internal config service; described in talks. | ✅ Verified — Airbnb Engineering blog |
| **LinkedIn** | Cluster Manager and dynamic config. | ✅ Verified — LinkedIn Engineering blog |
| **HashiCorp customers** | Consul + Vault widely deployed. | ✅ Verified — HashiCorp case studies |
| **Most Kubernetes-based shops** | ConfigMap + Secret + external secrets operator. | ✅ Industry standard |
| **AWS-native shops** | Parameter Store + Secrets Manager. | ✅ Industry standard |
| **Spring/Java shops** | Spring Cloud Config very popular. | ✅ Industry standard |

---

## Further reading

- The Twelve-Factor App — III. Config.
- Microsoft Azure — External Configuration Store pattern.
- microservices.io — Externalized Configuration.
- HashiCorp Consul and Vault documentation — the canonical OSS stack.
- *Kubernetes in Action* / *Programming Kubernetes* — ConfigMap and Secret deep dives.
- AWS Well-Architected papers on configuration management.
- Netflix Tech Blog — Archaius and dynamic configuration.
- Facebook's "Configerator" engineering posts.
- *Cloud Native DevOps with Kubernetes* — practical configuration management.

---

*Diagram sources: [`../diagrams/src/externalized-config.d2`](../diagrams/src/externalized-config.d2), [`../diagrams/src/config-spectrum.d2`](../diagrams/src/config-spectrum.d2).*
