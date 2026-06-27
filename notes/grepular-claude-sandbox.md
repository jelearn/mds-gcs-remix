# Grepular Claude Sandbox — Design Notes

Source: [`../grepular-claude-sandbox/`](../grepular-claude-sandbox/)
Upstream: https://gitlab.com/grepular/claude-sandbox

---

## What It Is

A single Python 3.12+ script (`claude_sandbox.py`, ~5,400 lines, stdlib only) that wraps
the Claude CLI in an ephemeral Podman container. It is installed by copying or symlinking
the script into `$PATH` as `claude`, after which it behaves transparently as a drop-in
replacement for the real `claude` binary.

The key design insight: **per-project config lives in `.claude-sandbox.toml`**, checked into
each project repo. Every project gets exactly the sandbox it needs — different base images,
package sets, mounts, credential rules — with zero global configuration to manage.

---

## Architecture

```
Host filesystem
    .claude-sandbox.toml      ← per-project config (never exposed to container)
    ~/.claude/                ← Claude state (mounted into container)
    ~/.claude/plans/<project> ← isolated per-project plan dir (auto-created/cleaned)

claude_sandbox.py (host)
    ├── parse config
    ├── generate Dockerfile deterministically
    ├── hash Dockerfile → image name grepular/claude-{hash12}
    ├── build image if missing/stale
    ├── [optional] launch mitmproxy sidecar container
    ├── podman run --network=container:<sidecar> (or standalone)
    └── exec claude CLI inside container

Host daemon (for host_command / sidecar_command)
    unix socket ← shim inside container
    └── spawns host process or podman sidecar
```

### Image naming

Images are named `grepular/claude-{hash12}` where the hash is a 12-character SHA256 prefix
of the generated Dockerfile content. Different configs produce different images; identical
configs reuse the cached image. A config change that affects only runtime behavior (mounts,
network) does not rebuild the image.

### Ephemeral vs persistent

Containers are one-shot and `--rm` by default. Each `claude` invocation spins up a fresh
container and tears it down on exit. There is no long-lived daemon process. State that
persists across runs lives in the host-mounted `~/.claude/` directory.

---

## Configuration System (`.claude-sandbox.toml`)

The config file drives every aspect of the sandbox. The script hides the real config file
from the container — it mounts a blank read-only placeholder at the config path — so Claude
cannot read injection rules, credentials, or sandbox configuration.

### Build-time options (trigger image rebuild on change)

| Option | Purpose |
|--------|---------|
| `base_image` | Override default (`debian:13-slim`); must be Debian-based |
| `extra_packages` | Additional `apt-get` packages |
| `extra_packages_pip` | Additional `pip` packages |
| `extra_packages_go` | Go packages via `go install` |
| `copy_from` | Multi-stage copy from another image (e.g., pull a binary from upstream) |
| `path_prefix` | Paths prepended to container `$PATH` |
| `run` | Arbitrary `RUN` steps in the Dockerfile |
| `startup_background` | Background processes launched before Claude at container start |
| `podman` | Install Podman + podman-compose; mount host Podman socket |

### Runtime-only options (no rebuild)

| Option | Purpose |
|--------|---------|
| `network` | `slirp4netns` (default, isolated), `slirp4netns:loopback`, `host` |
| `max_age` | ISO 8601 duration; triggers rebuild when image exceeds this age |
| `mount` | Additional bind mounts (`src`, `dst`, `writable`, `mkdir`) |
| `env` | Explicit env vars (supports `${VAR}` interpolation from host env) |
| `env_passthrough` | fnmatch globs against host env vars to pass through |
| `hide` | Paths to shadow with blank read-only placeholders inside the container |
| `permission_mode` | `default`, `allowEdits`, `bypassPermissions` |

### Variable expansion

Path-typed fields support `${VAR}` interpolation against the host environment at parse time.
Unset references are hard launch errors — values are never silently substituted with `""`.
`~` expansion happens at use time and composes with `${VAR}` expansion.

---

## Key Features

### 1. Header-Injection Proxy (`[[proxy.inject]]`)

The flagship security feature. When one or more `[[proxy.inject]]` rules are configured, the
script launches a `mitmproxy/mitmproxy` sidecar container. The Claude container joins the
sidecar's network namespace (`--network=container:<name>`), so `127.0.0.1:<port>` on the
shared loopback is mitmproxy. Standard proxy env vars (`HTTPS_PROXY`, etc.) are set so
well-behaved HTTP clients route through the sidecar automatically.

The sidecar resolves credentials **on the host** — either via `${VAR}` from host env, or by
running a shell command — and injects them as HTTP headers, query-string parameters, or
WebSocket frame substitutions **after** the request leaves the Claude container. The
credential value never enters the container's filesystem, process env, or mounted config.

**Injection targets:**
- `header` — HTTP request header (most common: `Authorization`)
- `qs` — URL query-string parameter(s)
- `ws_replace` — placeholder substitution in first WebSocket client frame

**Matching rules support:**
- `host` — fnmatch glob(s) on hostname
- `port`, `path`, `exclude_path` — optional refinement
- `method` — restrict to specific HTTP methods (read-only enforcement)
- `graphql_operation`, `graphql_operation_name`, etc. — GraphQL-aware filtering for
  APIs that share a single `POST /graphql` endpoint for reads and writes

**Dynamic credentials:**
- `value = "${VAR}"` — resolved at launch from host env
- `command = "..."` — lazy host shell command, cached for sandbox lifetime by default
- `command_ttl` — refresh on ISO 8601 schedule
- `command_format = "json"` — script returns `{"value": "...", "ttl": ...}` to self-report TTL
- `retry_on_status`, `retry_on_body` — force-refresh and replay on auth failure

**Defense-in-depth (scrubbing):**
Response bodies and server→client WS frames are scanned for the injected credential and
redacted before reaching Claude. Scoped per-request — only responses to injected requests
are scanned.

**Threat model:**
- Protected: filesystem reads, env dumps, process inspection of the credential
- Protected (defense-in-depth): literal-bytes echo in response bodies
- Not protected: a hostile client inside the container connecting direct to `:443` bypassing
  `HTTPS_PROXY` loses the injected header but no secret leaks

The shared-netns approach means no host ports are published — the proxy is on a private
loopback visible only between the two containers.

### 2. Host Command Forwarding (`[[host_command]]`)

Whitelists host binaries so Claude can invoke them from inside the sandbox. A shim in the
container connects over a unix socket to a host-side daemon, which spawns the real binary
with configured argv prefix, env, and cwd.

**Why this matters:** host-side state (`.psqlrc`, `.pgpass`, kubeconfig, kerberos tickets,
SSH agent) never enters the sandbox. Claude can run `psql` against a host database without
any credentials being copied into the container.

**Safety controls:**
- `allow_extra_args = false` (default) — Claude cannot pass any arguments; the shim runs
  only the configured `command` and rejects anything extra with exit code 126
- `env` / `env_passthrough` — fine-grained control over which host env vars reach the subprocess
- Point `command` at a wrapper script to strip dangerous capabilities (e.g., a `psql-readonly`
  wrapper that forces `--context=staging-ro`)

**Transparent streaming:** stdin/stdout/stderr streamed bidirectionally including PTY
allocation for interactive tools; signals (SIGINT, SIGTERM, SIGWINCH) forwarded.

**Warning:** a whitelisted command runs on the host with the user's full privileges. This is
a deliberate hole in the sandbox. `psql` can `COPY ... FROM PROGRAM`. `git` can run hooks.
`kubectl` with cluster-admin can read every secret. Understand the blast radius before
adding an entry.

### 3. Sidecar Commands (`[[sidecar_command]]`)

Like `[[host_command]]` but the command runs in an isolated Podman container rather than
directly on the host. Useful for tools that need something unavailable in the sandbox image
(a Firefox cookie store, a specific build chain) without giving that tool host-level access.

**Key distinction from `[[host_command]]`:** sidecar mounts are visible only inside the
sidecar — not mirrored into the Claude container. So you can give `yt-dlp` read-only access
to `~/.mozilla/firefox` without ever exposing the cookie store to Claude itself.

`persist = true` creates a long-lived sidecar container; invocations become `podman exec`.
Used for tools with expensive startup (Tor bootstrap, JVM warm-up) or shared state.

### 4. Wayland Forwarding

Mount `${XDG_RUNTIME_DIR}/wayland-0` (just the socket, not the whole runtime dir) to enable
headed browser tests (Playwright, Chromium) inside the sandbox with display on the host's
Wayland session. The Wayland security model prevents the container from snooping on other
host windows.

### 5. Host Browser Forwarding

`$BROWSER` inside the container points to a script that forwards URL-open requests to the
host via zenity approval dialog. Used for OAuth login — Claude's auth callback is proxied
back into the container seamlessly.

### 6. Image Auto-Rebuild

Images rebuild automatically when:
- A build-affecting config option changes (Dockerfile hash changes)
- `max_age` threshold exceeded
- `--sb-rebuild` flag passed
- The Dockerfile generation logic in the script itself changes

`DISABLE_AUTOUPDATER=1` is baked into the image so Claude's own in-container self-updater
is disabled — it would write to the ephemeral container and then be discarded, leaking
state into the host-shared `~/.claude`. Image updates come from explicit `claude update`.

### 7. Per-project Plan Isolation

`.claude/plans/` is auto-created in the project directory and mounted over the global
`~/.claude/plans/` inside the container, so one project's plans cannot bleed into another's.

---

## Testing

- Pytest suite in `tests/` (~20 test files)
- Covers: config parsing, Dockerfile generation, image management, duration parsing
- Makefile targets: `test`, `test-verbose`, `lint` (ruff), `typecheck` (mypy)
- CI via GitLab (`.gitlab-ci.yml`)

---

## Strengths

1. **Zero dependencies** — stdlib only; installs by file copy
2. **Per-project config** — each project brings its own `.claude-sandbox.toml`; no global
   configuration drift
3. **Credential injection without secret exposure** — the proxy sidecar is the cleanest
   known solution to "Claude needs to call authenticated APIs without seeing the tokens"
4. **GraphQL-aware filtering** — fine-grained auth rules for read-only vs read-write on
   APIs that share a single POST endpoint
5. **Dynamic credential refresh** — short-lived OAuth tokens refreshed automatically with
   TTL caching, retry-on-401 replay
4. **Defense-in-depth scrubbing** — response bodies are scanned for credential leakage
5. **Flexible host access** — `[[host_command]]` for host-state tools; `[[sidecar_command]]`
   for isolated tools that need their own environment
6. **Deterministic image naming** — content-addressed by Dockerfile hash; no stale images
7. **Config never exposed** — the real `.claude-sandbox.toml` is always hidden from Claude

## Gaps

1. **Ephemeral only** — not designed for persistent development sessions; no VS Code, no
   browser IDE, no display
2. **Honor-system proxy** — `HTTPS_PROXY` is set but not enforced at the network layer;
   a client that ignores it bypasses the proxy (though no secret leaks)
3. **No forcible network filtering** — unlike mitmproxy-dev-sandbox, there are no iptables
   rules forcing traffic through the proxy; this is acknowledged as future work
4. **No allowlist enforcement** — the proxy injects credentials but has no explicit
   domain/path deny-by-default policy (it injects where rules match; non-matching requests
   pass through without injection, not blocked)
5. **No multi-user isolation inside the container** — everything runs as one user (root by
   default in the container)
6. **Single-file size** — at ~5,400 lines, the single-file design is maximally portable
   but makes navigation and testing somewhat harder
7. **Podman-only** — Docker support was explicitly not pursued
