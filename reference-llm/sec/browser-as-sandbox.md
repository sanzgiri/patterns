# Browser-as-Sandbox

**Aliases:** browser-isolated agents, remote-browser agent execution, browser security boundary, web-as-isolation-layer
**Category:** Security & Containment
**Sources:**
[OpenAI — Introducing Operator (Jan 2025)](https://openai.com/index/introducing-operator/) ·
[Paul Kinlan — agent isolation writing (2024-2025)](https://paul.kinlan.me/) ·
[Anthropic — Computer use & sandboxes (Oct 2024)](https://www.anthropic.com/news/3-5-models-and-computer-use) ·
[Same-origin policy / browser security model docs (MDN, web.dev)](https://web.dev/secure/) ·
[Google Project Zero — browser security writeups](https://googleprojectzero.blogspot.com/)

---

## Problem

> [!TIP]
> **ELI5.** Building a sandbox for an agent is hard. Browsers already *have* one — 25+ years of battle-tested isolation: site-to-site isolation, no filesystem access, no shell, OS-level process sandbox. If your agent works *only inside a browser tab*, it inherits all of that for free. **Browser-as-sandbox** means: don't give the agent access to your computer; give it access to a browser, ideally a *remote* one. Whatever the agent does, it can only do as much as a normal user clicking around in a tab can do. This is the architecture choice behind ChatGPT Operator, several browser-use agents, and increasingly the recommended pattern for consumer agents that need to interact with the web.

The naïve agent architecture — agent runs on the user's machine with full OS access and shell — has the *biggest* possible blast radius. It can read files, change settings, install software, and access every credential the user has stored. Even with prompt-injection-aware defenses, this is hard to make safe.

A growing 2024-2026 architectural alternative: **don't put the agent on the user's machine at all**. Put it inside a browser, often a *remote* browser running in a cloud sandbox. The browser brings its own security model — site isolation, no filesystem, no shell — that the agent inherits automatically. The user watches the agent work via a video stream or shadow DOM; they don't grant the agent direct OS access.

Paul Kinlan argued this case publicly in late 2024 ("the browser is already a sandbox; use it"). OpenAI shipped Operator (January 2025) with this architecture. Anthropic's computer-use sandbox guidance is in the same spirit. By 2026, browser-as-sandbox is the recommended pattern for consumer-facing agents that need to act on web apps.

## How it works

> [!TIP]
> **ELI5.** Put the agent inside a browser running on a server (not your computer). The agent can navigate URLs, click, type, and read pages — exactly what a browser tab can do, nothing more. The browser's own security model isolates the agent from your computer, from cross-site attacks, and limits damage if the agent goes wrong. You watch what it's doing via a video stream.

![Browser-as-sandbox vs local agent](../diagrams/svg/browser-as-sandbox.svg)

### What the browser security model brings, for free

Decades of browser-security investment by Google, Mozilla, Apple, and Microsoft have produced a sophisticated isolation model:

- **Site isolation.** Each origin (scheme + host + port) runs in its own OS process. A compromise in one tab doesn't leak data from another tab on a different origin.
- **Same-origin policy / CORS.** Scripts on `siteA.com` can't read `siteB.com`'s data without explicit CORS permission.
- **No filesystem access.** Browsers can't read or write local files (with narrow exceptions like File System Access API, which require user permission).
- **No shell / process spawning.** A web page can't run arbitrary code on the user's machine.
- **OS-level process sandbox.** Beyond the browser's own isolation, each renderer process runs in an OS sandbox (seccomp on Linux, AppContainer on Windows, sandbox profiles on macOS).
- **Cookie / credential isolation per origin.** Cookies for `bank.com` aren't visible to scripts on `attacker.com`.
- **Permission prompts.** Camera, mic, location, notifications — each requires explicit user grant per origin.

For a *human* user, this model has held up reasonably well — most security incidents are user-action-driven (phishing, malware install) rather than browser-tab-breakouts. The hypothesis behind browser-as-sandbox: this same model can serve as the agent's containment.

### Two architectural variants

**Local browser.** Agent runs as a browser extension or attached browser-automation framework (Playwright, Selenium) controlling the user's actual browser.
- Pros: simpler setup, agent sees what the user sees, sessions are pre-logged-in.
- Cons: agent has access to *all* the user's open tabs and logged-in sessions → bigger blast radius. Any tab the user is logged into is reachable.

**Remote browser (preferred for safety).** Agent runs in a browser hosted in a cloud sandbox. The user watches a video stream and approves actions.
- Pros: maximum isolation. Agent has only the sessions the user explicitly logged into within the remote browser. Closing the session destroys the environment.
- Cons: latency. User must re-login to services. Some sites detect headless / remote browsers and block.

OpenAI's Operator chose the remote-browser variant explicitly for safety reasons. The user authorizes specific sites; the remote browser holds the session; closing Operator ends the session.

### What it doesn't fix

Browser-as-sandbox is *not* a complete defense. It's an architectural choice that limits *some* blast radius but leaves others:

- **Leg 2 of the lethal trifecta is still present.** Browser-agents read untrusted web content as their core function. Prompt injection from page content is fully possible.
- **Leg 1 is partially present** when the agent is logged into user accounts. The agent can read whatever those accounts can access (email, bank, social).
- **Leg 3 is also partially present** — the agent can take actions in those logged-in apps (send email, post, transfer money).
- **Cross-site within-allowlist attacks remain possible** — if the attacker can manipulate one site the agent is logged into to influence another.
- **Detection bypass** — sites with anti-bot measures may need to be unblocked, weakening sandboxing.
- **Account compromise via subtle interaction** — the agent can be tricked into clicking a phishing link, granting an OAuth scope, etc.

The architecture *reduces the surface area* dramatically compared to a full OS-access agent, but doesn't eliminate it. Production deployments still need per-action permission gating ([auto-mode permission boundaries](auto-mode-permission-boundaries.md)) and [maker-checker](../agt/maker-checker.md) for sensitive operations.

### Why remote browser specifically

Remote-browser-as-sandbox has properties local-browser can't match:

1. **The agent never touches the user's machine.** Even if the agent is fully compromised, nothing on the user's local filesystem or local apps is reachable.
2. **Disposability.** End the session, the entire environment is destroyed. No persistent state for an attacker to colonize.
3. **Network egress is the cloud provider's, not the user's.** Easier to put a strict egress allowlist around the cloud sandbox than around the user's machine.
4. **Audit / replay.** The video stream can be saved; every action attributed and replayable.
5. **Multi-user scaling.** The same architecture serves enterprises with hundreds of users; each gets their own remote browser session with their own credential scope.

The cost is latency, login friction, and engineering complexity. For consumer agents, it's increasingly considered worth it.

### Concrete production examples

**OpenAI Operator (Jan 2025).** Runs remote Chromium. User authorizes specific origins; user watches the live stream; user can intervene. The architecture choice was explicitly justified on safety grounds at launch.

**Anthropic Claude — computer use (Oct 2024).** Reference implementation runs Claude controlling a sandboxed VM with a browser. Anthropic explicitly recommends running in a dedicated sandbox, not on the user's machine.

**Browser-use, Stagehand, Manus, various consumer agents (2025-2026).** Mixed: some run locally, some remote, some hybrid. Trend is toward remote / sandboxed for the same safety reasons.

**Google's AI overviews / Gemini browser integrations.** Run in Google's infrastructure, not on user's machine.

**Microsoft Copilot for Browser.** Hybrid: read-only to the local browser, separate sandboxed agent for actions.

**Cognition Devin (for codebases hosted as web apps).** Runs in cloud sandbox with controlled access to the user's repos via OAuth scope, not full machine access.

### When the user *wants* local

The cost of remote browser is real, and some use cases push back:

- **Local files matter.** "Summarize this PDF on my desktop" requires local access.
- **Local credentials matter.** SSH keys, signed git commits, password managers — these are on the local machine.
- **Latency-sensitive tasks** like real-time tutoring or coding can't tolerate cloud round-trips.
- **Offline use** is impossible with remote browser.

For these cases, the alternative pattern is **local agent + sandboxed sub-process for risky actions**: the trust boundary moves from "the whole agent runs remotely" to "the agent runs locally but spawns sandboxed children for any work involving untrusted content." Less clean, but sometimes necessary.

### The trade-off table

| Property | Local agent (full OS) | Browser extension agent | Remote-browser agent |
|---|---|---|---|
| **Blast radius** | Whole machine | Browser + open tabs | Remote session only |
| **Filesystem access** | Yes | Limited (File System Access API) | No |
| **Shell / process spawning** | Yes | No | No (sandboxed) |
| **Cross-site protection** | None | Browser-native | Browser-native + cloud isolation |
| **Local credentials reachable** | Yes | Yes (via tabs) | No |
| **Disposability** | Hard (touches OS) | Medium (restart browser) | Trivial (end session) |
| **Latency** | Lowest | Low | Higher (cloud round-trip) |
| **Login friction** | None (uses existing logins) | None | Significant (re-login required) |
| **Site compatibility** | Best | Good | Variable (some block headless) |
| **Audit / replay** | Hard | Medium | Easy (video stream) |

For consumer agents acting on the web, the right column is the increasingly-recommended default.

### Architecture details that matter

**Visible video stream vs invisible.** Most production setups show the user what the agent is doing. This adds a *human-in-loop* signal — the user can intervene. Invisible / headless is more efficient but removes that safety net.

**Per-origin permission grants.** Operator requires the user to explicitly authorize each new site. This limits the credential exposure.

**Session lifecycle.** Sessions should be ephemeral by default — log out at end, destroy the browser profile. Long-lived sessions accumulate risk.

**OAuth scope granularity.** When the agent logs in via OAuth, request the *narrowest* scopes possible. Many providers offer read-only or per-resource scopes.

**Detection-bypass discipline.** Sites that block remote browsers (anti-bot) shouldn't be bypassed casually — that often means circumventing security measures the user wants. Better to fail honestly than to spoof.

## Variants & related patterns

- [**Containment & blast radius**](containment-blast-radius.md) — the umbrella pattern this implements.
- [**Lethal trifecta**](lethal-trifecta.md) — browser-as-sandbox breaks some legs partially.
- [**Egress allowlisting**](egress-allowlisting.md) — complementary; restrict what the remote browser can reach.
- [**Auto-mode permission boundaries**](auto-mode-permission-boundaries.md) — gate sensitive actions even in browser.
- [**Maker-checker**](../agt/maker-checker.md) — human approval for irreversible actions.
- [**Computer use**](../agt/computer-use.md) — most computer-use agents adopt browser-as-sandbox or full-VM-as-sandbox.
- **Site isolation** (browser engineering) — the underlying browser-internal mechanism.
- **Headless browsers** (Playwright, Puppeteer, Selenium) — typical implementation tooling.

## When NOT to use

- **Tasks requiring local-OS access.** File system, local apps, shell.
- **Performance-critical real-time tasks** where cloud latency is unacceptable.
- **Offline use.**
- **When the agent only acts on a single internal service** — running the agent inside that service's deployment may be simpler and more secure than a browser sandbox.

For pure web-action use cases (consumer agents that book travel, fill forms, navigate apps), it's now the default recommendation.

## Implementations

| Tool / platform | Browser-as-sandbox support |
|---|---|
| **OpenAI Operator** | Native — remote Chromium with site authorization |
| **Anthropic computer use** | Reference sandbox includes browser; works with VMs |
| **Playwright + cloud sandbox** | DIY building block |
| **Browserbase, Hyperbrowser, AnchorBrowser** | Hosted browser-as-sandbox-as-a-service |
| **Browser-use (OSS)** | Local Chromium with agentic control |
| **Stagehand** | Browser automation framework with agentic primitives |
| **E2B, Modal sandboxes** | Can host a sandboxed browser |
| **Chrome DevTools Protocol, CDP** | Low-level building block |
| **Microsoft Edge for Business + Copilot** | Enterprise browser with agent integrations |

## Companies / projects exemplifying browser-as-sandbox

- **OpenAI** ✅ — Operator launched Jan 2025 with explicit safety reasoning.
- **Anthropic** ✅ — computer-use sandbox guidance.
- **Browserbase, Hyperbrowser** ✅ — productize hosted browser sandboxes.
- **Manus, browser-use (OSS), Stagehand** ⚠ — implement variants of the pattern.
- **Microsoft Copilot for Browser** ⚠ — hybrid model.
- **Google Gemini browser integrations** ⚠ — server-side execution model.
- **Adept (acquired by Amazon)** ⚠ — built early action-oriented agents with similar isolation principles.
- **Paul Kinlan** ✅ — argued publicly for this architecture choice; widely cited.

## Further reading

- [Introducing Operator](https://openai.com/index/introducing-operator/) — OpenAI Jan 2025
- [Computer use & sandbox setup](https://www.anthropic.com/news/3-5-models-and-computer-use) — Anthropic Oct 2024
- [Paul Kinlan — agent isolation arguments](https://paul.kinlan.me/) — running blog
- [Browser security model overview](https://web.dev/secure/) — Google web.dev
- [Same-origin policy (MDN)](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)
- [Site Isolation design doc (Chromium)](https://www.chromium.org/Home/chromium-security/site-isolation/)
- [Browser-use OSS project](https://github.com/browser-use/browser-use)
- [Embrace The Red — browser-agent attack writeups](https://embracethered.com/blog/) — concrete attack examples

---

*Diagram source: [`../diagrams/src/browser-as-sandbox.d2`](../diagrams/src/browser-as-sandbox.d2)*
