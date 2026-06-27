# Design Notes

This directory contains design notes for the `mds-gcs-remix` project — an exploration of
combining the best elements of two sandbox projects:

- [`mitmproxy-dev-sandbox`](../mitmproxy-dev-sandbox/) (`mds`) — containerized VS Code + mitmproxy dev environment
- [`grepular-claude-sandbox`](../grepular-claude-sandbox/) (`gcs`) — single-file Python tool for running Claude CLI in Podman

The submodules are read-only references. Nothing in them is modified here.

## Contents

| File | Purpose |
|------|---------|
| [grepular-claude-sandbox.md](grepular-claude-sandbox.md) | Deep analysis of the gcs project: architecture, features, strengths, gaps |
| [mitmproxy-dev-sandbox.md](mitmproxy-dev-sandbox.md) | Deep analysis of the mds project: architecture, features, strengths, gaps |
| [comparison.md](comparison.md) | Side-by-side comparison and synthesis opportunities |

## Reading Order

For humans new to the project: start with the comparison, then dive into the individual notes
for depth. For AI agents: read all three before proposing any design work.
