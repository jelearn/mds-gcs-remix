# Conversation: Open-Source Projects Similar to MITMProxy Dev Sandbox

A discussion about finding open-source projects comparable to [`jelearn/mitmproxy-dev-sandbox`][mds],
evaluating the landscape against a prioritized feature set (drawn from the [`mds-gcs-remix` design
notes][remix-notes]), and deciding whether to improve the project, treat it as a niche, or contribute
elsewhere — with licensing as a key constraint.

Assistant: Claude Opus 4.8 (Anthropic)

It also contains additional information added after further reading.

---

## Turn 1

### User

> Can you find any open source projects similar to: https://github.com/jelearn/mitmproxy-dev-sandbox

### Assistant

[`mitmproxy-dev-sandbox`][mds] is a containerized sandbox for running AI coding agents (like Claude
Code) safely. It combines podman containerization, [mitmproxy][] to intercept and allowlist all
outbound HTTPS traffic, and a browser-based VS Code ([code-server][]) streamed via [noVNC][] for
isolation — so no JavaScript from inside the container executes on the host OS.

Similar projects, grouped by how closely they match the original's core combination of
*container + mitmproxy egress allowlist + AI agent*:

**Closest matches (mitmproxy + container + egress allowlist for agents)**

- [**mattolson/agent-sandbox**][agent-sandbox] — a mitmproxy sidecar enforces allowed hosts plus
  request-aware rules and blocks non-matching traffic with 403, while an iptables firewall blocks all
  direct outbound from the agent container so nothing bypasses the proxy. Supports Claude Code and
  other agents via CLI or devcontainers. Probably the single closest analog.
- [**Airut `airut-sandbox`**][airut-sandbox] (from the [haulos.com writeup][airut]) — runs each task in a rootless
  Podman container on an internal network with no direct internet access, routing all HTTP(S)
  transparently through mitmproxy enforcing a domain allowlist, plus a custom DNS responder that
  blocks DNS exfiltration. Same Podman + mitmproxy foundation, oriented toward CI/GitHub Actions.

**AI agent sandboxes with an egress firewall (different proxy tech)**

- [**schmitthub/clawker**][clawker] — a self-hosted CLI that runs Claude Code and other agents in
  isolated Docker containers, pairing a deny-by-default egress firewall (Envoy + custom CoreDNS +
  eBPF) with image building, monitoring, and parallel git-worktree agents.
- [**Infisical/agent-vault**][agent-vault] — a MITM credential proxy and vault for agents that does
  egress filtering and can run in strict deny mode, rejecting unmatched hosts with 403.
- [**luckyPipewrench/pipelock**][pipelock] — an open-source AI agent firewall that scans mediated
  HTTP/MCP/WebSocket traffic for exfiltration, SSRF, and prompt injection.

**Agent container/VM sandboxes (isolation-focused, lighter on the proxy)**

- [**textcortex/spritz**][spritz]: Spritz is a self-hosted control plane for spawning
  isolated agent instances on Kubernetes.  Mentioned because it's a successor to:
    - [**textcortex/claude-code-sandbox**][cc-sandbox] — runs Claude Code in
      isolated Docker (or Podman) containers with a browser-based terminal.
      (Archived; continued as "Spritz.")
- [**streamingfast/sbox**][sbox] — a Docker sandbox/MicroVM wrapper for Claude Code and OpenCode that
  bridges your `~/.claude` config into the isolated environment.
- [**webcoyote/clodpod**][clodpod] — runs AI agents (Claude Code, Codex, Gemini, Cursor Agent)
  isolated inside a macOS virtual machine.

On the noVNC / browser-streamed angle specifically, the closest is [OpenClaw's sandbox][openclaw],
which offers password-protected noVNC observer access with short-lived token URLs for sandboxed
browser sessions — though it's a full agent runtime rather than a "VS Code in a browser" dev box.

Two curated lists worth bookmarking, since the space moves fast:
[**bureado/awesome-agent-runtime-security**][awesome-runtime] and
[**gmh5225/awesome-ai-security**][awesome-aisec].

---

## Turn 2

### User

> Having a browser-based IDE is not a strict requirement, however some level of OS sandboxing and
> network control is mandatory. This site https://github.com/jelearn/mds-gcs-remix/tree/main/notes
> (that attempts to combine the MITMProxy Dev Sandbox and Grepular Claude Sandbox) provides a number
> of desirable features that should be prioritized.
>
> My current objective is to understand what other projects I can either learn from to improve the
> MITMProxy Dev Sandbox Project, understand if my idea is fulfilling a niche use case, or whether it
> might be worthwhile contributing to another open source project.
>
> Beyond that, license requirements are important as I would like anything I work on to be able to be
> used with little restrictions (possibly MIT or at most a GPL that only requires sharing the source
> code, like GNU Affero General Public License v3.0)

### Assistant

The [`mds-gcs-remix` notes][remix-notes] define a precise feature target. The defining combination
being aimed for is: **forcible kernel-level interception (iptables REDIRECT) + credential injection
without secret exposure + per-project config + optional persistent IDE + multi-user isolation**,
packaged permissively.

[Grepular's `claude-sandbox`][gcs] is **AGPLv3** (created February 2026) — within the stated ceiling.

#### License + feature scorecard

The license column was verified directly from each repo; "has / lacks" is relative to the target
combo (forcible interception + credential injection without exposure + per-project config + optional
IDE + multi-user isolation).

| Project | License | Has (vs. target) | Lacks |
|---|---|---|---|
| **[your `mds`][mds]** | MIT-0 | iptables-forced interception, deny-by-default allowlist, persistent browser IDE, multi-user | credential injection, per-project config, rich rules, tests |
| **[grepular `gcs`][gcs]** | AGPLv3 | credential injection w/o exposure, per-project config, rich/GraphQL rules, dynamic creds, tests | forcible enforcement, IDE, multi-user |
| **[mattolson/agent-sandbox][agent-sandbox]** | MIT | iptables + mitmproxy sidecar, scheme/method/path/query rules, devcontainers | credential injection, IDE, multi-user |
| **[Infisical/agent-vault][agent-vault]** | MIT (core; `ee/` commercial) | credential injection w/o exposure, egress filtering, multi-tenant, dynamic creds | iptables-forced interception, IDE, it's a network proxy not a dev sandbox |
| **[schmitthub/clawker][clawker]** | AGPLv3 | deny-by-default egress firewall, worktree parallelism, credential *forwarding* | injection-without-exposure, IDE, multi-user-in-container |
| **[luckyPipewrench/pipelock][pipelock]** | Apache-2.0 (core; `enterprise/` ELv2) | egress firewall, exfil/injection scanning, audit receipts | injection, IDE, it's a firewall layer not a sandbox |
| **[webcoyote/clodpod][clodpod]** | Apache-2.0 | strong VM isolation, multi-agent | network allowlist depth, injection, IDE |

[`streamingfast/sbox`][sbox] and [`textcortex/claude-code-sandbox`][cc-sandbox] have no LICENSE file
(and the latter is archived), so both should be treated as not safely reusable.

#### 1. What to learn from (by feature axis)

**Credential injection without exposure** — three techniques worth comparing, since gcs's
header-injection is the simplest but not the most robust. [Infisical/agent-vault][agent-vault] is
purpose-built: it injects credentials on outbound HTTPS via a MITM proxy so agents never see them,
with multi-tenancy and strict deny mode. The [Airut sandbox][airut] uses a stronger variant — masked
secrets are replaced with format-preserving surrogate tokens that the proxy swaps for real values
only on scoped hosts, and AWS signing credentials are re-signed by the proxy so the secret key never
enters the container. Surrogate-token masking closes a leak path plain header-injection doesn't.

**Solving the honor-system bypass** (synthesis opportunity #1 in the notes) — [Airut][airut] is the
cleanest reference: all HTTP(S) is routed transparently through mitmproxy with no `HTTP_PROXY` env
vars needed, and a custom DNS responder returns the proxy's IP for all queries and never forwards
upstream, blocking DNS exfiltration. That transparent-redirect + captive-DNS pattern handles the
Go/Java/curl clients that ignore `HTTPS_PROXY`.

**Richer allowlist + per-request rules** — [mattolson/agent-sandbox][agent-sandbox]'s mitmproxy addon
checks decrypted requests against scheme, method, path, or query rules and 403s non-matches, with an
iptables firewall blocking all direct outbound. That's the exact enforcement model plus the rule
richness to add — and it's MIT, so freely borrowable.

**Threat-model docs + "credentials live outside the sandbox" architecture** — Anthropic's
open-source [sandbox-runtime / secure-deployment guidance][secure-deploy] describes a
`--network none` container whose only egress is a mounted Unix socket to a host proxy that enforces
domain allowlists and injects credentials, so a prompt-injected agent can't exfiltrate to arbitrary
servers. Essentially the productionized form of what gcs does, and a good model for the threat-model
writeup the notes flag as missing. See also Anthropic's [Claude Code sandboxing post][cc-sandboxing]
and the [sandbox-environments comparison][sandbox-envs].

#### 2. Is this a real niche?

Yes — the specific *union* is unfilled. Every individual capability exists somewhere, but no
permissively-licensed project combines forcible kernel-level interception **and** credential injection
without exposure **and** per-project config **and** a persistent IDE **and** multi-user isolation. The
closest siblings each miss two or three.

The genuinely rare differentiator is the **persistent browser IDE with pixel-streamed isolation** —
almost nobody else does this; the field is converging on ephemeral CLI/devcontainer flows. The only
adjacent thing is [OpenClaw's][openclaw] password-protected noVNC *observer* for browser sessions,
which isn't a dev IDE.

Honest caveat: that same IDE is what *narrows* the audience. Most demand is for headless,
parallelizable, CI-friendly agent runs, which is why so many MIT/Apache projects cluster there. The
IDE is simultaneously the moat and the niche-limiter. The notes already hedge this by proposing to
make the display/IDE optional via config — treat that as the strategic pivot: an enforcement+injection
core that *can* run headless, with the IDE as a first-class but optional layer.

#### 3. Contribute, or build?

Three different moves (more than one is possible):

- **Best "join forces" candidate: [mattolson/agent-sandbox][agent-sandbox] (MIT).** Closest
  philosophical sibling to mds — same mitmproxy+iptables enforcement model, compatible permissive
  license, complementary strengths (it has rule richness and devcontainer ergonomics; mds has the
  IDE, pixel-streaming, and multi-user isolation).
- **Best well-scoped upstream contribution: the iptables-enforcement piece back into [gcs][gcs].**
  Literally synthesis opportunity #1 in the notes; self-contained; fixes gcs's acknowledged biggest
  weakness. AGPLv3 is fine for an upstream PR.
- **Don't contribute, do build — for the IDE vision.** Mike scoped gcs deliberately as
  ephemeral/CLI and flags the host-exposure features as out-of-scope; [agent-vault][agent-vault] is a
  network-deployed proxy, not a local dev sandbox. Neither is a home for "persistent IDE + forced
  interception + multi-user."

**Licensing decision that's load-bearing:** if gcs code is *vendored* into the remix, AGPLv3 is viral
and the whole combined work becomes AGPLv3. If mds stays the base and the injection concept is
*reimplemented independently* (the mitmproxy addon API is public; the idea isn't copyrightable, only
Mike's code is), the whole thing can stay MIT/MIT-0. Since maximum permissive reuse is a stated goal,
the clean-room reimplementation path preserves the most downstream freedom; reserve AGPL for the
narrow case of genuinely lifting gcs source.

---

## Hightlights / Inspiration / Thoughts

- General:
    - I should be looking to easily choose which LLMs (local or remote) the
      agents use.
    - A lot of the other projects have very nice CLIs that seem to provide
      the right level of control over the sandbox.
    - Really need to decide:
        - How many platforms do I really want to support?
        - Stay with Podman?  Support it and Docker?  Find another solution that
          provides a common layer for interaction across platforms?
    - Compared to some of the projects, I like the idea of keeping the host
      as pristine, requiring minimal dependencies or alterations, and being very
      deliberate with what gets put in and pulled out of the sanbox.
    - Relying on Kernel-level security was the minimal, light-weight level of
      trust I was willing to go with, but the concept of MicroVMs as another
      layer is interesting, but will require more research.
        - Having the agent able to spin-up its own containers (with same project
          restrictions, is appealing).

- `agent-sandbox`:
    - Uses [Lima][lima] to support macOS, which is interesting as it also supports
      multiple backends and will run on Linux, which could address the cross-platform
      aspirations for this project.
    - Aims to support multiple editor integrations, which is interesting but
      not sure if that should also be an objective.
    - Seemingly broad support for multiple agents, which is definitely a stretch goal.
        - Supports agent switching vs. having multiple available, and separate
          state volumes for history/credentials/etc.
    - Similar mitmproxy and firewall solution, but it takes it further by
      injecting secrets in the mitmproxy sidecar container so the agent doesn't
      see the credentials.  This is interesting and potentially worth exploring.
        - It also includes support for github access token injection.
    - The agent sandbox includes an agent skill to help avoid having the agent
      fight against the sandbox it's in, which is an interesting concept.
    - It supports per-project configurations, which I think is the way to go.
        - Also, per agent policy configuration support is interestig as well.
    - Specifically calls-out using a debian container with pinned dependencies
      for reproducable environments, which is something that really should be
      considered.

- `clawker`:
    - Interesting in that it adds monitoring into the setup (as an option).
    - Takes extra steps with egress firewall and DNS protectiosn which would be
      interesting to explore.
    - Seems to provide similar credential forwarding as others, but extends it
      further for SSH/GPG forwarding, host browser forwarding.
        - I'm not sure how much host access I'd like to support, ideally only
          the minimum, but there might be some useful software use worth exploring.
    - Host vs. Guest interactions:
        - The "Bind or snapshot workspace" feature is worth looking into further
          as this situation already exists with `mds` as a slightly annoying
          edge case.
        - This includes fresh/copy agent mode that with either start without
          any agent config, or copy in all your Claude Code settings.
            - Interesting, but most likely need to scope how much is copied in
              by default and make it obvious.
        - Environment variables from host to guest possible, but I think keeping
          it to a minimum (or at least only via a project configuration) would
          be ideal.
    - The use of a shared docker network is interesting for agents, and the
      software used is most likely good to explore: Envoy + custom CoreDNS + eBF firewall
    - Uses and interesting package labeling technique with `pkg/whail` (Whale
      Jail) to only control specific Docker resources related to the project.
    - Features like Static Dockerfile generation, injectable build-time
      dependencies, and post initialization scripts is interesting.
        - This is worthwhile looking into further and seeing how best to support
          bespoke per project/workspace customizations (like "profiles" in
          `sbox`)
    - The CLI is rich, and has nice shortcuts and alias options for convenience
      for common actions.
        - Includes robust resource management of the Docker resources it creates.
    - It's another example of the AGPL-3.0 usage license, but has parts that
      are MIT license and has an interesting Contributor License Agreement.

- `sbox`:
    - Uses Docker, not Podman... is there a need to support either?
    - Supports [Docker sandbox (MicroVM)][docker-sandbox] which is an interesting concept for
      another layer of isolation with (as in other projects) the abiity for
      the sandbox to run its own internal containers.  (feature only for macOS,
      & Windows, & no Podman equivalent)
    - It would be interesting to support Windows, macOS, as well as Linux.
    - Profiles for different tooling to install is an interesting concept and
      aligns with existing objectives.
    - Also supports different agents, but seemingly independencly vs. at same
      time on the same workspace/sandbox.  (is this better? simpler? desirable?)
    - Has ability to inject host OS environment variables into container, not
      sure if this is something to support.
    - Attempts to read Claude code agent, plugins, and CLAUDE/AGENT .md files
      from parent directories, not sure if that's something that aligns with
      the project.
    - Also supports sandbox state, etc. per project, though I will be looking
      to find opportunities for selective re-use across projects.
    - The CLI seems nice and robust for all elements that need interaction.

- `clodpod`:
    - Interesting it supports so many agents, though limited to macOS.
    - Is VM based (good/bad) so it requires managing them individually for
      each project/workspace (incl. memory/cpu budgets).
        - The global management CLI features seem nice, but I prefer the per
          project/workspace config method.
    - Is also bash-based (does the job), but I'm still not sure if I'd like
      to continue down that path.
        - Though it uses shellspect (testing framework) and custom shellcheck
          config, which could be good to learn from.

- `claude-code-sandbox` and `spritz`:
    - While the `claude-code-sanbox` is now archived, it can still be used
      for inspiration conceptually:
        - Simple CLI.
        - Project configuration elements highlight similar challenges already
          facing `mds` (e.g. docker image, file, environment, volumes, mounts, etc).
            - Having already run into these issues initially and solving them
              using podman compose files, then backing out for simplicity,
              potentially it would be good to revisit how much control should
              be exposed.
    - `spritz` seems to have taken the concept to the next level for deployments
      in Kubernetes, which is most likely out of scope as the initial use case
      was more human scale, local development with agents.
        - However the concept of having sandboxed long-running agents is
          interesting and having them sandboxed would still be a requirement.
        - For now a single-user focus is the priority over larger scale
          organization-size deployments.

---

## References

### Primary projects

[mds]: https://github.com/jelearn/mitmproxy-dev-sandbox
[remix-notes]: https://github.com/jelearn/mds-gcs-remix/tree/main/notes
[gcs]: https://gitlab.com/grepular/claude-sandbox
[agent-sandbox]: https://github.com/mattolson/agent-sandbox
[agent-vault]: https://github.com/Infisical/agent-vault
[clawker]: https://github.com/schmitthub/clawker
[pipelock]: https://github.com/luckyPipewrench/pipelock
[spritz]: https://github.com/textcortex/spritz
[cc-sandbox]: https://github.com/textcortex/claude-code-sandbox
[sbox]: https://github.com/streamingfast/sbox
[clodpod]: https://github.com/webcoyote/clodpod

- MITMProxy Dev Sandbox (`mds`): <https://github.com/jelearn/mitmproxy-dev-sandbox>
- mds-gcs-remix design notes: <https://github.com/jelearn/mds-gcs-remix/tree/main/notes>
- Grepular Claude Sandbox (`gcs`): <https://gitlab.com/grepular/claude-sandbox>
- mattolson/agent-sandbox: <https://github.com/mattolson/agent-sandbox>
- Infisical/agent-vault: <https://github.com/Infisical/agent-vault>
- schmitthub/clawker: <https://github.com/schmitthub/clawker>
- luckyPipewrench/pipelock: <https://github.com/luckyPipewrench/pipelock>
- textcortex/claude-code-sandbox: <https://github.com/textcortex/claude-code-sandbox>
- streamingfast/sbox: <https://github.com/streamingfast/sbox>
- webcoyote/clodpod: <https://github.com/webcoyote/clodpod>

### Curated lists

[awesome-runtime]: https://github.com/bureado/awesome-agent-runtime-security
[awesome-aisec]: https://github.com/gmh5225/awesome-ai-security

- bureado/awesome-agent-runtime-security: <https://github.com/bureado/awesome-agent-runtime-security>
- gmh5225/awesome-ai-security: <https://github.com/gmh5225/awesome-ai-security>
- FlorianBruniaux/claude-code-ultimate-guide: <https://github.com/FlorianBruniaux/claude-code-ultimate-guide>

### Articles & documentation

[airut]: https://haulos.com/blog/sandboxing-github-actions/
[airut-sandbox]: https://github.com/airutorg/airut
[openclaw]: https://docs.openclaw.ai/gateway/sandboxing
[secure-deploy]: https://code.claude.com/docs/en/agent-sdk/secure-deployment
[cc-sandboxing]: https://www.anthropic.com/engineering/claude-code-sandboxing
[sandbox-envs]: https://code.claude.com/docs/en/sandbox-environments
[docker-sandbox]: https://docs.docker.com/ai/sandboxes/
[lima]: https://github.com/lima-vm/lima

- Airut — Sandboxing AI-Authored Code in GitHub Actions: <https://haulos.com/blog/sandboxing-github-actions/>
- OpenClaw — Sandboxing: <https://docs.openclaw.ai/gateway/sandboxing>
- Anthropic — Securely deploying AI agents (Agent SDK): <https://code.claude.com/docs/en/agent-sdk/secure-deployment>
- Anthropic — Making Claude Code more secure with sandboxing: <https://www.anthropic.com/engineering/claude-code-sandboxing>
- Claude Code Docs — Choose a sandbox environment: <https://code.claude.com/docs/en/sandbox-environments>
- INNOQ — Restricting network access for AI coding agents with a proxy allowlist: <https://www.innoq.com/en/blog/2026/03/dev-sandbox-network/>
- OpenAI — Codex agent approvals & security: <https://developers.openai.com/codex/agent-approvals-security>
- Modal — Run Claude Code in a Modal Sandbox: <https://modal.com/docs/examples/sandbox_agent>
- EveryDev — claude-sandbox (Superset/Docker): <https://www.everydev.ai/tools/claude-sandbox>

### Supporting components

[mitmproxy]: https://mitmproxy.org/
[code-server]: https://github.com/coder/code-server
[noVNC]: https://github.com/novnc/noVNC

- mitmproxy: <https://mitmproxy.org/>
- code-server: <https://github.com/coder/code-server>
- noVNC: <https://github.com/novnc/noVNC>
- StefanMaron/claudeCodeAlDevContainer: <https://github.com/StefanMaron/claudeCodeAlDevContainer>
