# Hightlights from Grepular Claude Sandbox for MDS

Interesting design/implementation take-aways from Grepular Claude Sandbox:

- Single-executable deployment pattern (vs. repo cloning).
    - Incl. Python probably more maintainable/flexible vs. Bash
- Per-workspace config file for per-project adjustments.
    - Incl. better proxy rule configuration that is more easily ajusted per project.
    - This is great to quickly spinning-up temporary/adhoc workspaces.
    - Incl. custom side-car background processes per project.
- Much more control over additional guest OS dependencies (incl. python, go, etc.)
- Better documentation, in particular:
    - Threat model overview.
    - How Claude updates are handled.
- Container image re-use across projects is useful.
    - Incl. better naming vs. local prefix
- Container age-off rules is useful.
- It takes special care for local project workspace `.claude` directories, which
  will conflict with the container's.  It's actually important that usage within
  or without the sandbox use cases (based on preference) is supported.
- HTTP/S Proxy with header injection, this could be useful to reduce the level
  of trust given of guest user account.
- Actually has tests and a build pipeline.

Other inspiration:

- Provide ability to re-use read-only mounts for Agent configs and MITM Proxy?
    - i.e. re-use accounts when desired

Conflicting design choices:

- Has ability to expose more of the host OS than desired.
    - e.g. to host wayland, browser, etc.
    - that level of acces is out of scope
- From a source perspective, having Python embeded within a Python script is
  less ideal from a review/observability perspective.
