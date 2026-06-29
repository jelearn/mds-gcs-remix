# Design Notes

This directory contains design notes for the `mds-gcs-remix` project — an exploration of
combining the best elements of two sandbox projects:

- [`mitmproxy-dev-sandbox`](../mitmproxy-dev-sandbox/) (`mds`) — containerized VS Code + mitmproxy dev environment
- [`grepular-claude-sandbox`](../grepular-claude-sandbox/) (`gcs`) — single-file Python tool for running Claude CLI in Podman

The submodules are read-only references. Nothing in them is modified here.

## Contents

| File | Purpose |
|------|---------|
| [comparison.md](comparison.md) | Side-by-side comparison and synthesis opportunities |
| [highlights-gcs.md](highlights-gcs.md) | Covers high-level improvements for MITMProxy Dev Sandbox based on Grepular Claude Sandbox's design and implementation. |
| [research-other-projects.md](research-other-projects.md) | Initial research on other projects for inspiration or collaboration |
| [grepular-claude-sandbox.md](grepular-claude-sandbox.md) | Deep analysis of the gcs project: architecture, features, strengths, gaps |
| [mitmproxy-dev-sandbox.md](mitmproxy-dev-sandbox.md) | Deep analysis of the mds project: architecture, features, strengths, gaps |

## Reading Order

For humans new to the project: start with the comparison, then dive into the other  notes
for depth. For AI agents: read all referenced files before proposing any design work.
