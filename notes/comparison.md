# Comparison and Synthesis Opportunities

This document compares the two submodule projects and identifies where a combined design
could capture the best of both.

---

## Side-by-Side Summary

| Dimension | grepular-claude-sandbox (gcs) | mitmproxy-dev-sandbox (mds) |
|-----------|------------------------------|------------------------------|
| **Primary use case** | Ephemeral Claude CLI invocations | Persistent development sessions |
| **Interface** | CLI (`claude` drop-in) | Browser-based VS Code via VNC |
| **Config model** | Per-project `.claude-sandbox.toml` | Single shared config per clone |
| **Credential injection** | Yes — proxy sidecar, secrets never enter container | No — credentials must enter `coder`'s env |
| **Network enforcement** | Honor-system (`HTTPS_PROXY`) | Kernel-level iptables REDIRECT |
| **Proxy filtering** | No deny-by-default; injects where rules match | Deny-by-default allowlist (exact hostname + path prefix) |
| **Allowlist model** | N/A (no blocking) | `(hostname, path_prefix)` tuples, Python file, hot-reload |
| **Auth injection rules** | Rich: fnmatch, method, GraphQL operation, dynamic TTL, retry | None |
| **Multi-user isolation** | No (single user in container) | Yes (coder / mitm / display) |
| **Host command access** | Yes (`[[host_command]]`) | No |
| **Sidecar containers** | Yes (`[[sidecar_command]]`) | No |
| **Dynamic credentials** | Yes (command, TTL, retry-on-401) | No |
| **Display / IDE** | No | Yes (code-server, Chromium, VNC/noVNC) |
| **Allowlist editing** | N/A | Edit Python file + `manage.sh reload-allowlist` |
| **Testing** | Pytest suite, ruff, mypy, GitLab CI | shellcheck, pylint; no automated tests |
| **Implementation** | Single Python file (~5,400 lines), stdlib only | Bash scripts, Python addon, Containerfile |
| **Podman** | Required (>= 4.9.3) | Required (>= 4.9.3) |
| **Installation** | File copy, symlink as `claude` | Clone repo, run `./manage.sh start` |

---

## Core Tension

The two projects solve adjacent problems but make opposite trade-offs in the same dimensions:

**gcs** maximizes per-project flexibility and credential safety at the cost of enforcement:
the proxy is honor-system, there is no deny-by-default policy, and the container has no
persistent IDE.

**mds** maximizes enforcement and session quality at the cost of flexibility: traffic is
forcibly intercepted but there is no way to inject credentials safely, and every project
shares the same allowlist and configuration.

A combined design could have both: **forcible interception (mds) + credential injection
without secret exposure (gcs) + per-project config (gcs) + persistent IDE (mds)**.

---

## Feature Gap Matrix

### What gcs has that mds lacks

| Feature | Why it matters |
|---------|----------------|
| Per-project `.claude-sandbox.toml` | Different projects need different packages, mounts, and auth rules; a single shared config is friction |
| Credential injection without secret exposure | Claude can call authenticated APIs without secrets entering `coder`'s environment; currently mds has no safe path for this |
| Dynamic credential refresh (TTL, retry-on-401) | Short-lived OAuth tokens need periodic refresh; currently mds has no mechanism for this |
| GraphQL-aware filtering | Some APIs use `POST /graphql` for both reads and writes; method filtering is insufficient |
| Rich allowlist matching (fnmatch, method, path exclusions) | mds's exact-hostname + path-prefix model cannot express `*.github.com` or method restrictions |
| Host command forwarding | mds has no equivalent; tools that need host-side state (kubeconfig, `.pgpass`) can't be used without copying credentials in |
| Sidecar containers | mds has no concept of isolated per-tool containers |
| Automated test suite | mds has no automated tests; gcs has Pytest + ruff + mypy |

### What mds has that gcs lacks

| Feature | Why it matters |
|---------|----------------|
| Kernel-level network enforcement (iptables) | gcs's honor-system proxy can be bypassed by any HTTP client that ignores `HTTPS_PROXY`; mds's REDIRECT cannot be bypassed by `coder` |
| Deny-by-default allowlist | gcs injects credentials but does not block non-matching requests; Claude could still reach arbitrary hosts, just without injected tokens |
| Persistent IDE (VS Code + VNC) | gcs is ephemeral; mds supports ongoing development sessions with a full browser-based IDE |
| Pixel-streamed display | No container JavaScript reaches the host; gcs has no display at all |
| Multi-user isolation (coder/mitm/display) | gcs runs everything as one user; mds separates the agent, proxy, and display at the Linux user level |
| `verify-users` confirmation | Auditable process-ownership check; gcs has no equivalent |
| `proxy-log` / `blocked` / `allowed` monitoring | Live visibility into what the agent is accessing; gcs has no equivalent |
| Per-sandbox multi-instance model | Cloning the repo gives a fully independent environment; gcs is per-invocation |

---

## Synthesis Opportunities

The following are ordered roughly by impact and feasibility.

### 1. Enforce the proxy at the iptables layer in gcs

**Problem:** gcs sets `HTTPS_PROXY` env vars but cannot force all traffic through the
proxy. A client that ignores `HTTPS_PROXY` (common in Go, Java, some curl invocations) will
bypass injection and reach the upstream unauthenticated.

**Solution:** adopt mds's iptables REDIRECT approach inside the container. The gcs sidecar
already shares a network namespace with the Claude container; adding iptables REDIRECT rules
inside that netns would force all outbound :443/:80 through the proxy, matching the strength
of mds.

**Complexity:** medium — requires the container to run with `CAP_NET_ADMIN` to set iptables
rules. The sidecar container already exists; the rules just need to be applied.

### 2. Add credential injection to mds

**Problem:** mds intercepts all traffic but has no mechanism to inject credentials. Claude
must receive tokens directly in its environment (defeating isolation) or go unauthenticated.

**Solution:** port gcs's `[[proxy.inject]]` mechanism into mds's `allowlist.py` addon.
The mitmproxy Python addon API supports request modification natively — adding a
header-injection layer alongside the allow/block logic is straightforward.

Credentials would be read from a config file on the host (outside the container) and
injected by the mitmproxy addon running as `mitm`. The `coder` user would never see the
values.

**Complexity:** medium — `allowlist.py` already runs as `mitm` and has full mitmproxy API
access. The main work is the config format and credential resolution (reading from host env,
running host commands, TTL caching).

### 3. Per-project configuration in mds

**Problem:** mds uses a single global allowlist and a single `.env` file. Different projects
need different allowed domains, different injected credentials, and potentially different
packages.

**Solution:** introduce a per-project config file (e.g., `.sandbox.toml`) analogous to
gcs's `.claude-sandbox.toml`. The `manage.sh` script reads the project's config at startup
and merges it with a base config.

For the allowlist specifically, a merged model works well: a base allowlist (Anthropic API,
npm, PyPI, GitHub) is always present; per-project entries extend it.

**Complexity:** medium — `manage.sh` and `allowlist.py` need to support config merging.
Defining the config schema is the main design decision.

### 4. Richer allowlist rules in mds

**Problem:** mds's `ALLOW_RULES = list[tuple[str, str]]` model is limited: exact hostname
matching only, path prefix only, no method filtering.

**Solution:** upgrade to a richer rule model inspired by gcs:
- Hostname: fnmatch glob (e.g., `*.github.com`, `*.anthropic.com`)
- Path: prefix or fnmatch glob with optional exclude
- Method: optional restriction to specific HTTP methods

This can be done by extending `allowlist.py` without changing the interface — existing
entries continue to work as-is (exact match is a subset of fnmatch).

**Complexity:** low-medium — pure Python change to `allowlist.py`; no infrastructure changes.

### 5. Allowlist CLI / REPL (TODO item in mds)

**Problem:** editing `allowlist.py` and running `reload-allowlist` while watching `proxy-log`
is friction. The mds TODO explicitly calls this out.

**Solution:** a thin CLI (e.g., `./manage.sh allow <hostname> [path]` /
`./manage.sh block <hostname>`) that appends or removes entries from a JSON config file that
`allowlist.py` reads alongside its static rules. Hot-reload continues to work via the
file-watch poller.

This is cleaner if the rule format is JSON rather than Python (avoiding the need to parse
and edit Python source), which also supports opportunity #3 (per-project config).

**Complexity:** low-medium.

### 6. Pre-bake extensions into the mds image (TODO item in mds)

**Problem:** VS Code extensions install at first startup, making the initial session slow.

**Solution:** bake commonly-used extensions into the Containerfile using
`code-server --install-extension` during build. Per-project extensions could be installed
at container start from a per-project extension list.

**Complexity:** low.

### 7. Automated tests for mds

**Problem:** mds has no automated test suite. shellcheck and pylint run, but no behavioral
tests verify that the firewall rules actually enforce the isolation properties or that the
allowlist correctly blocks/allows requests.

**Solution:** a minimal Pytest suite (analogous to gcs's) that:
- Verifies the allowlist logic in isolation (unit tests on `_is_allowed`)
- Verifies firewall rule generation logic
- Integration tests (container-up + `curl` attempts from `coder` to blocked/allowed hosts)

**Complexity:** low for unit tests; medium for integration tests.

---

## Overall Assessment

Neither project is strictly better. They target different operating modes (ephemeral
one-shot vs. persistent session) and make complementary trade-offs (flexibility vs.
enforcement). The most impactful synthesis path is:

1. **Start with mds** as the base (persistent IDE, forcible enforcement, multi-user isolation)
2. **Port gcs's credential injection** into `allowlist.py` — this is the highest-value
   missing feature and the most self-contained addition
3. **Upgrade the allowlist rule model** to fnmatch + method filtering — low-effort, high
   value for real-world usage
4. **Add a per-project config file** — once #2 is in place, per-project injection rules
   become essential; they need a home

Opportunities #1 (force enforcement in gcs) and #5–#7 (allowlist CLI, extension pre-baking,
tests) are valuable but lower priority relative to closing the credential-injection gap.
