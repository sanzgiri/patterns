# Feature Flag (Feature Toggle)

**Aliases:** Feature Toggle, Feature Switch, Feature Gate, Conditional Feature, Runtime Toggle
**Category:** Operations / Release
**Sources:**
[Pete Hodgson — *Feature Toggles (aka Feature Flags)*, martinfowler.com (2017)](https://martinfowler.com/articles/feature-toggles.html) — the definitive article ·
*Continuous Delivery* (Humble & Farley, 2010) — Ch on branching strategies ·
[LaunchDarkly documentation](https://docs.launchdarkly.com/) ·
[Unleash documentation](https://docs.getunleash.io/) ·
[GrowthBook documentation](https://docs.growthbook.io/) ·
[Microsoft FeatureManagement (.NET)](https://learn.microsoft.com/en-us/azure/azure-app-configuration/feature-management-overview)

---

## Problem

> [!TIP]
> **ELI5.** You want to release new code, but you're not sure it's ready for everyone. Maybe the UI isn't polished, maybe it works for English but not Japanese, maybe you want only paying customers to see it, maybe you want to A/B test it. Without feature flags, your only choice is "deploy it = everyone sees it" or "don't deploy yet." With **feature flags**, you deploy the code with the feature switched *off*, and turn it on (for specific users, regions, %, or everyone) using a runtime config — no redeploy. Want to turn it back off? Flip the flag. Deploy and release become two independent decisions.

The fundamental observation is that **deploying code** and **releasing a feature to users** are different concerns:

- **Deploy** = the code is running in production.
- **Release** = users experience the new behavior.

Conventional CI/CD conflates them — when you deploy, users see the change. This creates several problems:

- **Half-finished features** force long-lived feature branches (with all their merge pain), because incomplete code can't go to production.
- **Region/segment-specific rollouts** require maintaining separate code paths or separate deployments.
- **Quick rollback** of a single feature requires reverting a code change and redeploying — minutes-to-hours, not seconds.
- **A/B testing** requires separate infrastructure to run two variants.
- **Operational kill switches** (turn off the expensive recommendation engine during a traffic spike) require redeploys.

Feature flags decouple these by introducing a runtime conditional:

```python
if feature_flag('new_checkout_flow', user=current_user):
    new_checkout(user)
else:
    old_checkout(user)
```

The flag's value is determined at runtime by a configuration service that the application queries. Toggling the flag enables/disables the feature for chosen users *without code change*.

This sounds trivial. The implications are surprisingly deep — Pete Hodgson's 2017 martinfowler.com article became the canonical reference, and an entire industry of vendors (LaunchDarkly, Split, Unleash, GrowthBook, ConfigCat, Optimizely, AWS AppConfig, etc.) exists around this single pattern.

## How it works

> [!TIP]
> **ELI5.** Deploy your code with new feature behind an `if` statement. Have your app ask a flag service "is feature X on for user Y?" The flag service has rules ("on for 5% of users in EU", "on for all internal employees", "off in production"). To launch the feature, you flip a switch in the flag dashboard — the change reaches your app in seconds. To roll it back, flip the switch off.

### Architecture

The system has three parts: the application code, an SDK in the application that evaluates flags, and a management service with rules:

![Feature flag architecture](../diagrams/svg/feature-flag.svg)

The flow:

1. The application has a conditional that asks the flag SDK: "is `new_checkout` on for this user?"
2. The SDK has a local cache of flag rules (so the question is answered in microseconds, not by a network call).
3. The SDK evaluates the rule against the user context (user ID, plan, region, A/B bucket).
4. The rule returns true or false (or a variant: A vs B vs C for multivariate).
5. The application executes the appropriate code path.

Separately, a management service hosts the dashboard, audit log, and rule editor. When rules change, the new rules are pushed (WebSocket, SSE) or polled to the SDKs. Propagation is seconds.

### Targeting capabilities

Real flag services support sophisticated targeting:

- **Percentage rollouts**: "5% of users get the new feature." The bucketing is deterministic per user (hash of user ID) so the same user consistently sees the same variant.
- **User attributes**: "users with plan = enterprise", "users created after 2024-01-01", "internal employees".
- **Segments**: predefined groups (beta-testers, friends-of-team, paid customers).
- **Region / locale**: "on in US, off in EU until GDPR review."
- **Multivariate**: not just on/off but A vs B vs C (for A/B/C testing).
- **Time-based**: scheduled enable/disable.
- **Dependencies**: "feature X only on if feature Y is on."
- **Sticky / canary**: gradual ramp by percentage with monitoring.

This is what makes feature flags more than just an `if config.enabled` — they're a sophisticated targeting and experimentation system.

### The four kinds of feature flag (Hodgson's taxonomy)

Pete Hodgson's seminal article identifies four distinct use cases, with different lifetimes and owners. Treating them all the same is the most common mistake:

![Feature flag use cases](../diagrams/svg/feature-flag-uses.svg)

**1. Release Toggles** — ship in-progress features dark, enable when ready. Short-lived (days to weeks). Owned by the dev team. **Must be removed** once fully rolled out — leaving them creates code complexity.

**2. Experiment Toggles** — A/B test variants to measure conversion / retention / engagement. Lifetime of weeks to months (until statistical significance). Owned by product/data. Removed after a winner is picked.

**3. Ops Toggles (Kill Switches)** — disable a feature during an incident; throttle expensive functionality under load. Long-lived (always there). Owned by ops/SRE. Treated as live operational tools.

**4. Permission / Entitlement Toggles** — feature is permanently available to some users (enterprise plan, internal users, beta program). Permanent. Owned by product. Essentially business rules implemented via the flag system.

Mixing these is dangerous: an experiment flag with no removal date becomes flag debt; an ops kill switch with the wrong default becomes a hidden incident waiting to happen.

### The flag debt problem

The single biggest anti-pattern: **flags that are never removed**. Each release toggle, once at 100% rollout, should be deleted (along with the old code path). In practice, flags accumulate:

- Feature shipped successfully → "I'll remove the flag later" → never happens.
- Both code branches must be maintained — every test, every refactor.
- Tens of thousands of flags pile up; teams lose track of which are still meaningful.
- New developers can't tell if a code branch is dead or "behind a flag that's been on for 2 years and we forgot."

Treatments:
- **Track a removal date on every release flag.**
- **Quarterly cleanup sweeps** to remove fully-rolled-out flags.
- **Linting**: tools that detect flags older than N days and flag (no pun) them.
- **Stale flag reports** from the management vendor.
- **Treat the flag count as technical debt**; like dependency count, monitor it.

Mature orgs have flag governance: who can create flags, what the lifetime is, what the cleanup process is. LaunchDarkly even has a "stale flag" report built in.

### Performance and latency

Naive implementations call the flag service on every request — bad. Real SDKs:

- **Cache flag rules locally** in the app (refresh on push or poll every N seconds).
- **Evaluate locally** — microseconds, not network round trips.
- **Asynchronously emit metrics** ("this user got variant A") to the flag service for experiment analysis.
- **Handle disconnect gracefully** — default value if flag service unreachable.

Good SDKs add ≤100µs per flag evaluation. Bad implementations add tens of ms. The difference is whether you've cached or are making a network call.

### Server-side vs client-side flags

Two patterns with different security implications:

- **Server-side flags**: evaluation happens in the backend. Rules are private. Used for sensitive features (pricing logic, access control). The browser/app just sees the result.
- **Client-side flags**: rules are shipped to the browser/mobile app. User sees the rule logic. Used for UI changes, less-sensitive experiments. Can be tampered with — never use for security decisions.

A common mistake: using a client-side flag to gate access to a paid feature. The user can flip it in DevTools. Server enforcement (and ideally server-side feature flag check) is required for anything that affects authorization, billing, or data exposure.

### Experiment (A/B test) flags

A flag with multiple variants (A vs B), each shown to a deterministic % of users, with metrics emitted per variant, becomes an experiment platform:

```python
variant = feature_flag('checkout_v2_experiment', user)
if variant == 'control':
    old_checkout(user)
elif variant == 'redesigned':
    new_checkout(user)
elif variant == 'simplified':
    minimal_checkout(user)
emit_event('checkout_started', user, variant)
```

The flag service collects per-variant outcomes (conversion, revenue, time-to-complete) and reports statistical significance. This is **experimentation as a service**, built on the feature flag foundation.

Major platforms (LaunchDarkly Experimentation, Split, Statsig, GrowthBook) all combine flags + experiments. Many large companies build internal versions (Microsoft's ExP, Booking's experiment platform).

### Operational kill switches

A different use entirely: a flag that's permanently in the codebase, always evaluated, used to **disable functionality in an emergency**:

```python
if not feature_flag('enable_recommendation_engine'):
    return []  # skip expensive recommendations
recommendations = compute_recommendations(user)  # only when flag is on
```

These are long-lived ops tools. Common kill switches:
- Expensive features that can be disabled under load (recommendations, related-items, suggestions).
- Optional dependencies that can be bypassed if down (analytics, A/B tracking, third-party APIs).
- New code paths during initial rollout, with a "back to old code" fallback.
- Country-specific compliance toggles.

These are never removed. They're part of the operational toolkit. The ops team needs to know they exist and be empowered to flip them.

### Testing with flags

Flags multiply test surface area: every flag adds a code path. Strategies:

- **Test both branches** of release flags — the new feature and the fallback.
- **Test with all permanent flags ON** (your "intended future") and **all OFF** (the safe fallback).
- **Avoid testing every combination** (2^N grows fast); test relevant subsets.
- **Use flag-aware test harnesses** that can override flags per test.

### Compared to deployment-time configuration

A relevant question: why feature flags instead of just config files / env vars?

- **Runtime-changeable**: no restart, no redeploy. Critical for incident response.
- **Per-user targeting**: env vars are per-instance; flags are per-request.
- **Centralized management**: dashboard, audit log, approvals.
- **Experimentation**: variant assignment, metrics, statistical analysis built in.
- **Gradual rollouts**: percentage-based ramping.

Static config is fine for things that genuinely never change (DB connection strings). Anything that benefits from runtime change → flags.

### Trade-offs

Advantages:
- **Deploy ≠ release**: separate concerns, separate risk decisions.
- **Instant rollback** for individual features.
- **Targeting** without separate deployments.
- **Experimentation** built into the same primitive.
- **Operational control** without engineering involvement.

Disadvantages:
- **Code complexity**: every flag adds branches.
- **Test surface**: every flag doubles the relevant test paths.
- **Flag debt**: unchecked accumulation kills maintainability.
- **External dependency**: flag service downtime → which default does your app use?
- **Cost**: vendor pricing (LaunchDarkly et al. are not cheap at scale).
- **Distributed reasoning**: behavior now depends on runtime state in another system.

The vendor question: build vs buy. Internal flag systems are not hard (Etsy's open-sourced FeatureLib, Unleash is OSS) but require infra and tooling. Vendor services are easy to start with but get expensive and create lock-in. Most large engineering orgs build their own; small/medium use a vendor.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Release toggle** | Short-lived; hide in-progress features. |
| **Experiment toggle** | A/B test variants. |
| **Ops toggle / kill switch** | Long-lived operational control. |
| **Permission toggle / entitlement** | Per-user feature access (business rule). |
| **Server-side flag** | Evaluation on backend; rules private. |
| **Client-side flag** | Rules in browser; not for security. |
| **Multivariate flag** | 2+ variants, not just on/off. |
| **Percentage rollout** | Ramp by % of users. |
| **Targeted rollout** | By region / segment / attribute. |
| **Canary** ([page](blue-green-canary.md)) | Traffic-level progressive rollout; complementary. |
| **Externalized config** ([page](externalized-config.md)) | More general runtime config pattern. |
| **Branch by Abstraction** | Code-level alternative to flags for refactors. |

## When NOT to use

- **Truly small / experimental code** where the complexity exceeds the benefit.
- **Without a removal discipline** — flag debt will dominate.
- **Security gates** via client-side flags — trivially bypassed.
- **Performance-critical paths** with poor SDK design — flag eval becomes hot spot.
- **Without testing both branches** — the off-path will eventually break unnoticed.
- **As a substitute for proper branching/CI** — flags don't fix every problem.

---

## Real-world implementations

| Tool | Notes |
|---|---|
| **LaunchDarkly** | Industry leader; enterprise. |
| **Split** | Big at large orgs; strong experimentation. |
| **Statsig** | Newer; built by ex-Facebook ExP team. |
| **Unleash** | Open-source; self-hostable. |
| **GrowthBook** | OSS; popular with startups. |
| **ConfigCat** | Cheap, simple. |
| **Optimizely** | Originally A/B; expanded to flags. |
| **AWS AppConfig** | Native AWS; cheap if you're on AWS. |
| **Azure App Configuration / Feature Management** | Microsoft's. |
| **Firebase Remote Config** | Mobile-focused. |
| **PostHog** | OSS analytics + flags. |
| **Flagsmith** | OSS + cloud. |
| **Custom (in-house)** | Most large orgs (Netflix, Facebook, Google, Microsoft, Booking) have proprietary platforms. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Facebook / Meta** | Original feature-flag-at-scale; "Gatekeeper" service. | ✅ Verified — Engineering talks |
| **Google** | "Server-side experiment framework"; widely-documented in research. | ✅ Verified — Tang et al. KDD 2010 paper on Google experimentation |
| **Microsoft** | ExP platform for Bing, Office, Azure. | ✅ Verified — published research papers |
| **Netflix** | Heavy use; built proprietary tools alongside experimentation. | ✅ Verified — Netflix Tech Blog |
| **LinkedIn** | XLNT / experimentation platform. | ✅ Verified — engineering blog |
| **Booking.com** | Famous for running thousands of concurrent experiments. | ✅ Verified — Booking's "How experiments work" posts |
| **Uber** | Morpheus experimentation platform. | ✅ Verified — Uber Engineering blog |
| **Etsy** | Open-sourced FeatureLib; influential post on flags. | ✅ Verified — Etsy Code as Craft |
| **Spotify** | Used at scale; published practices. | ✅ Verified — Spotify Labs |
| **GitHub** | Uses Scientist (for refactor) + Flipper (for flags). | ✅ Verified — both OSS by GitHub |

---

## Further reading

- Pete Hodgson — *Feature Toggles (aka Feature Flags)*, martinfowler.com (2017) — the canonical reference.
- *Continuous Delivery* (Humble & Farley) — branching strategies chapter.
- *Trustworthy Online Controlled Experiments* (Kohavi, Tang, Xu, 2020) — the experimentation textbook.
- Etsy's "How and why Etsy uses feature flags" blog posts.
- GitHub's Scientist library — a related tool for safe refactoring.
- Google's "Overlapping experiment infrastructure" KDD 2010 paper — large-scale experiment design.
- *Accelerate* (Forsgren, Humble, Kim) — feature flags as part of high-performing-team practices.

---

*Diagram sources: [`../diagrams/src/feature-flag.d2`](../diagrams/src/feature-flag.d2), [`../diagrams/src/feature-flag-uses.d2`](../diagrams/src/feature-flag-uses.d2).*
