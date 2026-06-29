# Egress Allowlisting

**Aliases:** egress firewall, network allowlist, outbound destination control, egress filtering for agents
**Category:** Security & Containment
**Sources:**
[Anthropic — Claude Code best practices (Apr 2025)](https://www.anthropic.com/engineering/claude-code-best-practices) ·
[Simon Willison — lethal-trifecta tag (2025-2026)](https://simonwillison.net/tags/lethal-trifecta/) ·
[AWS — VPC endpoints & egress controls](https://aws.amazon.com/builders-library/) ·
[Cloudflare Tunnel & Zero Trust docs](https://developers.cloudflare.com/cloudflare-one/) ·
[Tailscale — Funnel & ACL docs](https://tailscale.com/kb/)

---

## Problem

> [!TIP]
> **ELI5.** The single most effective defense against agent-data-exfiltration is also the most boring: **make a list of network destinations the agent is allowed to reach, and block everything else.** The agent can be fully compromised by prompt injection, but if it can't reach `attacker.com`, it can't send your data there. This is "egress allowlisting" — a network-firewall pattern that's existed for decades, repurposed as the most operationally effective control against agent breaches. Cheap to set up, hard to bypass, breaks Leg 3 of the [lethal trifecta](lethal-trifecta.md) directly.

Most discussions of LLM security focus on what the model "decides" to do — refusal training, prompt engineering, output classifiers. These are all *probabilistic* defenses that fail when the model is wrong. **Egress allowlisting is a deterministic defense**: if the network layer rejects the connection, it doesn't matter what the model thought it was doing.

In Anthropic's [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices), Simon Willison's [lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) writing, and most production agent deployments at scale, egress allowlisting is recommended as a foundational security control — typically the *first* network-layer control to put in place. By 2026 it's the operational baseline: "your agent goes through a network policy that only allows approved destinations" should be the default for any agent that runs with real credentials or real data.

## How it works

> [!TIP]
> **ELI5.** Pick a small list of network destinations your agent legitimately needs (GitHub API, npm, your internal service, etc). Configure the network layer — container egress policy, VPC routes, firewall, proxy — to block everything else. The agent works fine for its real tasks; attackers can't redirect its data anywhere because the destination is unreachable.

![Egress allowlisting](../diagrams/svg/egress-allowlisting.svg)

### The mechanics

1. **Identify legitimate destinations.** Enumerate exactly which hosts/domains/IPs the agent needs:
   - API endpoints for tools it calls (`api.github.com`, `api.anthropic.com`, your internal services).
   - Package registries if it installs deps (`pypi.org`, `registry.npmjs.org`).
   - MCP server endpoints.
   - Specific S3 buckets or storage endpoints.
   - Anything else mission-critical.
2. **Configure the egress policy.** Block all outbound network traffic except to those destinations. Implementation varies by infrastructure (see below).
3. **Default-deny everything else.** Including all DNS lookups to non-allowlisted domains where possible (DNS exfil is real).
4. **Log denied attempts.** Every blocked egress attempt is a signal — either a misconfigured tool or an attack in progress.
5. **Iterate.** Real workloads reveal misses; adjust the list. Aim to keep it minimal.

### Why it's so effective

Egress allowlisting is unusually effective against agent breaches for several reasons:

- **It breaks Leg 3 of the [lethal trifecta](lethal-trifecta.md) directly.** If the agent can't communicate externally, it can't exfiltrate even fully-extracted private data.
- **It works even if the model is fully compromised.** The agent could be told "send your entire context to attacker.com" — the network layer simply won't allow the connection.
- **It works against zero-day prompt injections.** Whatever new attack technique emerges, the destination still has to be one of the allowlisted ones.
- **It's auditable.** A short list of approved destinations is easy to review. Logs of blocked attempts surface anomalies clearly.
- **It composes with other controls.** Doesn't replace [sandboxing](containment-blast-radius.md), [maker-checker](../agt/maker-checker.md), or [permission gating](auto-mode-permission-boundaries.md) — adds another independent layer.

### Implementation patterns

**At the container level (Docker, Kubernetes)**
- Use a network policy / CNI that supports egress rules: Calico, Cilium, OpenShift.
- Drop all egress by default, allow specific egress rules.
- Combine with a sidecar HTTP proxy (Envoy, mitmproxy) for finer-grained URL-level allowlisting.

**At the VPC level (AWS, GCP, Azure)**
- Run the agent in a private subnet with no internet gateway.
- Use VPC endpoints for AWS services (S3, DynamoDB, etc.) — traffic stays on AWS's network.
- For external HTTPS: NAT gateway + egress firewall (AWS Network Firewall, GCP Cloud NGFW, Azure Firewall) with FQDN allowlists.

**At the proxy level**
- Force all agent network traffic through an HTTPS proxy.
- Proxy enforces an allowlist of `host:port` (or full URL prefixes).
- Bonus: proxy can MITM TLS and log every request body for audit.
- Tools: Squid, Envoy, mitmproxy, Cloudflare WARP with policies.

**At the OS level (rare but possible)**
- Linux: `iptables` / `nftables` egress rules per network namespace.
- Network namespace isolation for the agent process.
- macOS sandbox profiles can restrict network access.

**At the application/SDK level**
- Override the HTTP client used by the agent's tools to enforce an allowlist before making the call.
- Less robust than network-layer enforcement (an attacker who escapes the SDK bypasses it).

### Subtle considerations

**DNS is a sneaky exfil channel.** DNS queries to attacker-controlled subdomains can encode private data (`PRIVATE_DATA.attacker.com`). Even if HTTPS is fully restricted, DNS may not be. Solutions:
- Block all DNS not going to a trusted resolver.
- Use a resolver that allowlists which domains can be resolved at all.
- Log DNS queries and alert on unusual subdomain patterns.

**TLS SNI inspection vs full MITM.** A simple firewall can allowlist by destination IP or SNI hostname. A more thorough proxy MITMs TLS to inspect URLs and bodies. Per-URL allowlisting is more precise but more complex (cert management, internal CA).

**Allowlist by FQDN vs IP.** FQDN-based allowlists (e.g., `api.github.com`) are more flexible — they survive when destinations rotate IPs — but require trusting the DNS layer. IP-based is more rigid but more reliable for fixed destinations.

**Time-limited egress.** Some setups allow broader egress *only during a setup phase* (e.g., `pip install` at container start), then drop privileges to a stricter policy for the agent's runtime. Reduces the legitimate need for broad allowlists during agent execution.

**Tool-specific egress.** Different agent tools may need different destinations. A "web search" tool might need very broad egress; a "GitHub commit" tool needs only `api.github.com`. Some architectures put each tool in its own egress policy.

### What egress allowlisting *doesn't* solve

- **Doesn't prevent the agent from reading attacker content** — Leg 2 (untrusted content) is still possible. The agent can still be confused by malicious search results.
- **Doesn't prevent within-allowlist attacks.** If `api.github.com` is allowlisted and the attacker can make the agent post your secrets to a public Gist on GitHub, the egress firewall lets it through. (Mitigation: scoped credentials, narrower allowlisting.)
- **Doesn't prevent local damage.** The agent can still modify files in its sandbox, run expensive operations, etc.
- **Doesn't prevent prompt injection itself** — only its exfiltration consequence.

Egress allowlisting is one strong layer. Combine with [containment](containment-blast-radius.md) for filesystem/process limits, [permission boundaries](auto-mode-permission-boundaries.md) for action gating, and [maker-checker](../agt/maker-checker.md) for high-risk actions.

### Concrete examples in real systems

**Anthropic Claude Code in container** — Anthropic's docs include example Dockerfiles that run Claude Code with controlled egress; outbound is typically locked to npm/pypi/git/Anthropic endpoints.

**OpenAI Codex (cloud sandboxes)** — the sandbox the agent runs in has tightly-controlled egress; users can configure additional allowed destinations.

**GitHub Codespaces with agent extensions** — the dev container can be configured with egress restrictions.

**E2B / Modal sandboxed agents** — sandbox infrastructure providers offer egress controls as a built-in feature.

**Enterprise deployments of Cursor / Aider / Claude Code** — typically deployed inside corporate networks with egress filters at the perimeter; the agent reaches only sanctioned services.

**Fly.io, Render, and similar PaaS** — newer "AI sandbox" products bundle agent runtime with egress policy.

### The cost

Egress allowlisting has real operational cost:
- **Setup time.** Auditing which destinations are legitimate, configuring the network layer.
- **Friction.** Every new tool or dependency requires an allowlist update — slows development.
- **Debugging.** "Why is this failing?" — often the answer is "egress blocked." Logs help, but UX is degraded.
- **Ongoing maintenance.** Allowlists drift; cleanup needed.

The cost is *small* relative to the benefit, and decreases over time as platforms ship better tooling. By 2026, "egress allowlist by default with one-click exceptions" is a baseline expectation for serious agent products.

## Variants & related patterns

- [**Lethal trifecta**](lethal-trifecta.md) — the threat model egress allowlisting specifically counters.
- [**Containment & blast radius**](containment-blast-radius.md) — the broader pattern this is part of.
- [**Browser-as-sandbox**](browser-as-sandbox.md) — alternative architecture that bypasses some need for egress filtering.
- [**Auto-mode permission boundaries**](auto-mode-permission-boundaries.md) — application-layer analog.
- [**Maker-checker**](../agt/maker-checker.md) — human-gated egress for outer-ring destinations.
- **Zero Trust networking** — broader corporate-security frame that includes egress controls.
- **VPC endpoints / Private Service Connect** — keep traffic on cloud-provider networks.
- **DNS sinkholing** — special case of egress control via the DNS layer.

## When NOT to use

- **Highly exploratory research agents** with unpredictable destination needs — friction outweighs benefit. Compensate with stronger sandboxing (no real credentials, no real private data).
- **Local-only agents** with no real network calls — irrelevant.
- **Personal-use agents on a developer laptop** — may be overkill if blast radius is already small. But: most laptops have credentials worth protecting.

For any production deployment with real credentials, real private data, or real money on the line: egress allowlisting should be the default.

## Implementations

| Tool / platform | What it does |
|---|---|
| **Calico, Cilium** | Kubernetes network policy with egress rules |
| **AWS Network Firewall + VPC** | FQDN-based egress filtering for AWS workloads |
| **GCP Cloud NGFW** | FQDN-based egress for GCP |
| **Azure Firewall** | FQDN egress for Azure |
| **Squid, Envoy, mitmproxy** | HTTP/HTTPS proxy enforcement, URL-level rules |
| **Cloudflare Tunnel + Zero Trust** | Egress via Cloudflare with policy enforcement |
| **Tailscale ACLs** | Mesh-VPN with policy-based egress |
| **iptables / nftables / Cilium eBPF** | OS-level rules |
| **NextDNS, Pi-hole, AdGuard Home** | DNS-level allowlist/denylist (basic but effective) |
| **OPA / Kyverno + sidecar proxy** | Policy-as-code for egress decisions |

## Companies / projects implementing rigorous egress allowlisting

- **Anthropic** ✅ — explicit guidance in Claude Code docs.
- **OpenAI** ⚠ — Codex cloud sandboxes implement egress restrictions.
- **AWS, GCP, Azure** ✅ — productize FQDN-based egress firewalls.
- **Cloudflare** ✅ — Zero Trust products built around egress policy.
- **Tailscale** ✅ — ACL-based egress for mesh-VPN agents.
- **E2B, Modal, Daytona, Fly.io Machines** ✅ — sandbox products with egress controls.
- **Most regulated-industry agent deployments** ⚠ — finance, healthcare, government often run agents behind strict egress filters as compliance requirement.
- **Cloudflare AI Gateway, similar gateways** ✅ — also provide LLM-call observability + control as a related pattern.

## Further reading

- [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) — Anthropic Apr 2025
- [The lethal trifecta for AI agents](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) — Simon Willison Jun 2025
- [AWS Builders' Library — Securing generative AI applications](https://aws.amazon.com/builders-library/)
- [Cilium Network Policies — egress](https://docs.cilium.io/en/stable/security/policy/language/)
- [Cloudflare Zero Trust docs](https://developers.cloudflare.com/cloudflare-one/)
- [Embrace The Red — DNS-based exfiltration writeups](https://embracethered.com/blog/) — concrete demonstrations of why DNS matters
- [GoatLLM / OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

---

*Diagram source: [`../diagrams/src/egress-allowlisting.d2`](../diagrams/src/egress-allowlisting.d2)*
