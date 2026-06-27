# mitmproxy Dev Sandbox — Design Notes

Source: [`../mitmproxy-dev-sandbox/`](../mitmproxy-dev-sandbox/)
Upstream: https://github.com/jelearn/mitmproxy-dev-sandbox

---

## What It Is

A containerized development environment that runs VS Code (via code-server), mitmproxy,
and a pixel-streamed display (Chromium + noVNC) inside a single Podman container. All
outbound HTTPS traffic from the `coder` user is forcibly intercepted by mitmproxy — not
honor-system, but enforced at the iptables layer.

The key design insight: **strict multi-user isolation inside the container** separates
the agent user (`coder`) from the proxy user (`mitm`) and the display user (`display`)
at the Linux user level. iptables grants the REDIRECT exemption only to `mitm` — `coder`
has no path around the proxy regardless of what it tries.

---

## Architecture

```
Host browser → localhost:6080
                    │ pixels + input only (noVNC / websockify)
         noVNC + websockify (display uid 1102)
                    │
         Xtigervnc :5900 (display)
                    │
          Openbox window manager (display)
          xhost +SI:localuser:coder
             ┌──────────┴──────────┐
      code-server:8080          Chromium
         (coder uid 1100)      (coder uid 1100)
             └──────────┬──────────┘
                        │ all outbound :443/:80
                  iptables REDIRECT → :8081
                        │
               mitmproxy :8081 (mitm uid 1101)
                        │
                 allowlist.py addon
                ┌────────┴────────┐
              ALLOW             BLOCK
         (upstream TLS)        (403)
```

### Users

| User | UID | Role | Network access |
|------|-----|------|----------------|
| `coder` | 1100 | VS Code, Chromium, Claude Code, terminals | None — ALL :443/:80 REDIRECT'd to mitmproxy |
| `mitm` | 1101 | mitmproxy only | Outbound :443/:80 (gated by `allowlist.py`) |
| `display` | 1102 | Xtigervnc, Openbox, noVNC/websockify | Loopback only (VNC + noVNC ports) |
| `_apt` | (system) | Package management | Outbound :443/:80 (direct) |

UIDs are set at build time via `--build-arg CODER_UID=...` / `--build-arg MITM_UID=...`
and looked up at runtime by name so scripts remain correct if UIDs are overridden.

`mitm` has `/usr/sbin/nologin`, owns only the mitmproxy process and its config directory,
and cannot write to the workspace or read the API key.

---

## Key Components

### `Containerfile`

Multi-stage build:
- Stage 1: code-server (downloaded via install script; version pinned or latest)
- Stage 2: mitmproxy (pip install into `/opt/mitmproxy-venv`)
- Stage 3: final image — Debian bookworm-slim, all services assembled

Ports and UIDs are all parameterized via `ARG`/`ENV`:
- `CODER_UID=1100`, `MITM_UID=1101`, `DISPLAY_UID=1102`
- `MITM_PORT=8081`, `VNC_PORT=5900`, `NOVNC_PORT=6080`, `CODESERVER_PORT=8080`

The mitmproxy CA cert is generated once outside the image into a named Podman volume
(`*_mitmproxy_ca`) and re-used across rebuilds — no re-generation on every build.

### `scripts/entrypoint.sh`

Container init (runs as root), then delegates to per-service scripts as the correct user:
1. Apply iptables rules (`firewall.sh`)
2. Start mitmproxy as `mitm` (`start-mitmproxy.sh`)
3. Start display services as `display` (`start-display.sh`)
4. Start code-server as `coder` (`start-code-server.sh`)
5. Grant `coder` X11 access via `xhost +SI:localuser:coder`

### `scripts/firewall.sh`

The security core. Key properties:

**INPUT chain:**
- Accept: loopback, noVNC port (6080), established/related
- Default policy: DROP

**nat OUTPUT:**
- RETURN loopback (127.0.0.0/8) — dev servers on localhost bypass proxy
- REDIRECT :443 → :8081 for `coder` uid only
- REDIRECT :80 → :8081 for `coder` uid only

**filter OUTPUT:**
- loopback ESTABLISHED/RELATED: ACCEPT (all users, return traffic)
- loopback, root: ACCEPT unrestricted (entrypoint setup)
- loopback :8081 TCP, mitm only: ACCEPT
- loopback :5900/:6080 TCP, display only: ACCEPT
- loopback :5900/:6080 TCP+UDP, coder: DROP (coder cannot reach display services)
- loopback 1024:65535 TCP+UDP, coder: ACCEPT (covers code-server, mitmproxy redirect target, dev servers)
- Non-loopback ESTABLISHED/RELATED: ACCEPT (return traffic for mitm's upstream connections)
- DNS to container resolver: ACCEPT for root, coder, mitm, _apt
- Outbound :443/:80 for mitm and _apt only: ACCEPT
- Default policy: DROP

**Why coder needs DNS:** hostname resolution (`getaddrinfo`) happens in userspace before
the TCP SYN is sent. If coder can't do DNS, hostname lookups fail before the NAT REDIRECT
even fires. `coder` resolves hostnames, the TCP connection is then REDIRECT'd to mitmproxy.

### `config/mitmproxy/allowlist.py`

A mitmproxy Python addon (script-addon, not a full proxy addon class hierarchy). It:
- Defines `ALLOW_RULES: list[tuple[str, str]]` — exact hostname + path prefix pairs
- Builds an `_INDEX: dict[str, list[str]]` for O(1) host lookup
- On every request: check `_is_allowed(host, path)`; if not, respond with 403

Hot-reload: mitmproxy's built-in script-addon file watcher polls every ~1 second. Running
`./manage.sh reload-allowlist` (`podman cp` into the container) updates the mtime, and
new rules are active within ~1 second with no mitmproxy restart.

Current allowed domains: Anthropic API, OpenCode.ai, npm, PyPI, GitHub, Open VSX.

**Current limitation:** rules are `(exact_hostname, path_prefix)` tuples only — no wildcard
hostnames, no HTTP method filtering, no fnmatch patterns.

### `manage.sh`

Host-side control script. Key commands:

| Command | Action |
|---------|--------|
| `build` | Build the container image |
| `start` | Build if needed, start container |
| `stop` / `restart` | Stop or restart |
| `claude` | Shell into container as `coder` with Claude Code |
| `coder` | Bash terminal as `coder` |
| `opencode` | OpenCode.ai agent shell |
| `proxy-log` | Live ALLOWED/BLOCKED feed from mitmproxy |
| `blocked` / `allowed` | Recent blocked/allowed requests |
| `reload-allowlist` | `podman cp` updated `allowlist.py` into running container |
| `verify-users` | Confirm process ownership (mitm owns mitmdump, coder owns code-server) |
| `firewall` | Print current iptables rules |
| `reset-ca` | Remove mitmproxy CA volume (regenerated on next start) |
| `load_workspace` | Load `./workspace/` into the container's workspace volume |

### `config/code-server/`

- `config.yaml` — bind to `0.0.0.0:8080`, auth disabled (access is controlled by the
  container + VNC layer, not by code-server's own auth)
- `settings.json` — default VS Code settings (trust, telemetry, etc.)

### `config/openbox/rc.xml`

Window manager config: code-server's Chromium window is undecorated and maximized by
default so the VNC viewport is fully occupied by the IDE.

---

## Display Architecture

```
Host browser
    └── http://localhost:6080/vnc.html?resize=remote&autoconnect=true
            │ WebSocket
       websockify (display, port 6080)
            │ VNC protocol
       Xtigervnc :5900 (display)
            │ X11 display :1
       Openbox (display)
            │
       code-server Chromium window (coder)
```

`resize=remote` in the noVNC URL scales the remote display to fit the browser window.
The pixel stream means **no JavaScript from the container ever executes on the host browser**
— the host only receives rendered framebuffer pixels plus mouse/keyboard input.

---

## Multi-Sandbox Support

Multiple independent sandboxes can run simultaneously by cloning the repo into different
directories. Each clone has its own `.env` with a unique `AGENT_SANDBOX_NAME` and
`AGENT_SANDBOX_PORT`, separate Podman volumes, and an independent `allowlist.py`.

---

## Strengths

1. **Forcible network interception** — iptables REDIRECT at the kernel level; no honor-system
   proxy env var. `coder` cannot bypass mitmproxy even by hardcoding IP addresses or using
   non-standard tooling that ignores `HTTPS_PROXY`
2. **Multi-user isolation** — strict separation of roles at the Linux user level inside the
   container; `mitm` cannot write to workspace; `coder` cannot make raw outbound connections
3. **Deny-by-default allowlist** — non-matching requests get 403; the default is closed
4. **Hot-reloadable allowlist** — no mitmproxy restart needed; file-watch polling picks up
   changes within ~1 second
5. **Pixel-streamed IDE** — no container JavaScript executes on host; VNC + noVNC provides
   a full browser-based VS Code experience with display isolation
6. **Persistent sandbox** — long-lived container with persistent workspace volume; suitable
   for ongoing development sessions, not just one-shot invocations
7. **Multi-stage Containerfile** — good layer caching; code-server, mitmproxy, and display
   stack are independent stages
8. **Parameterized build** — UIDs, ports, and versions are all `ARG`/`ENV`; easy to
   customize without editing scripts
9. **Verified separation** — `./manage.sh verify-users` checks that each process is running
   as the correct user
10. **Well-structured linting requirements** — AGENTS.md mandates shellcheck (zero warnings)
    and pylint (≥9.0/10) before every commit

## Gaps

1. **No per-project credential injection** — the sandbox intercepts and can inspect traffic,
   but has no mechanism to inject auth tokens on behalf of the agent. Credentials must be
   passed into `coder`'s environment directly (defeating the purpose of keeping them off the
   agent) or proxied via some other mechanism
2. **Coarse allowlist model** — rules are `(exact_hostname, path_prefix)` tuples; no
   wildcard hostnames, no HTTP method restrictions, no GraphQL-aware filtering, no
   fnmatch patterns. The TODO acknowledges the desire for a richer rule format
3. **Allowlist editing is friction** — edit a Python file, run a shell command to reload.
   The TODO envisions a CLI/REPL interface for live add/remove while monitoring the proxy log
4. **No dynamic credential refresh** — no concept of TTL-based token refresh, `command`-based
   secret resolution, or retry-on-401
5. **Extensions not pre-baked** — VS Code extensions install at first startup (slow); they
   could be pre-baked into the image for faster startup
6. **code-server GitHub login prompt** — since 4.96.0 an unavoidable GitHub login prompt
   appears; current workaround is to pin to 4.109.5
7. **Clipboard not forwarded** — copy/paste between container and host OS not yet implemented
8. **Workspace loading friction** — on SELinux systems the `./workspace` directory cannot be
   directly mounted; `manage.sh load_workspace` is a manual copy workaround
9. **No per-project config** — all projects share the same allowlist, image, and configuration;
   there is no `.sandbox.toml` equivalent for per-project customization
10. **Podman-only** — Docker support explicitly not planned
