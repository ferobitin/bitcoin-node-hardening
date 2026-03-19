# Native Bitcoin Core & Core Lightning Node Architecture  
(Tor-Connected, systemd-hardened)

## Prerequisites

- Basic Linux (terminal, filesystem, editor)
- Systemd (understand, activate and debug units)
- Basic network understanding (SOCKS, Proxies, Tor)
- Bitcoin fundamentals (what a bitcoin full node is, what a Lightning node is and does)

## Overview

This repository contains hardened systemd units and configuration files for a
Bitcoin Core and Core Lightning setup, along with explanations.

The setup uses systemd and runs Tor-only. Execution is native on a Debian server
without Docker or VM abstraction layers.

This is an experimental learning environment to gain understanding of service interactions
and debugging skills on a hardened node setup.
Also, it is about gaining experience in running stable and controlled services
(observability, dependencies, resource limits, backup strategies).

## Architecture

The main components are [Bitcoin Core](https://github.com/bitcoin/bitcoin), [Core Lightning](https://github.com/ElementsProject/lightning), [Tor](https://www.torproject.org/) and [systemd](https://systemd.io/).

Core Lightning communicates with Bitcoin Core via JSON-RPC
and depends on Bitcoin Core at startup for blockchain access.

Tor provides outbound network transport via a local SOCKS5 proxy
as well as onion service exposure for inbound connections.

systemd defines service dependencies, the runtime environment and
controls service lifecycle behavior.

## Why this Way

The overall goal of this design is to serve as a learning-focused environment.

Transparency and debuggability are intended to understand the architecture under the hood
and to gain operational experience (uptime, limits, logs, state, backups, troubleshooting).

See this as a space to train comprehension, discipline and resource awareness.

## Repository Layout

Each service is documented separately (including design decisions and hardening considerations)
within a shared context.
```
/bitcoind/      - unit, bitcoin.conf, README
/lightningd/    - unit, config, README
/tor/           - unit, torrc, README
/environment/   - wrapper scripts, README
```

## Design Principles

- Native-first execution
- Explicit service isolation
- Minimal external dependencies (no dashboards)
- Observability over convenience (DIY, no Docker or Blackboxes)
- Understanding before automation (no Snap or Flatpak)

## Hardening

Hardening is applied at the service level (systemd sandboxing directives and namespace isolation
where supported, minimal read-write paths), user- and filesystem level (dedicated non-privileged users)
and network level (Tor-only, UFW restrictions).

The intention is to protect the system: minimize the attack surface and isolate permissions
(reduce blast radius).

## Out of Scope

This repository is not a tutorial, not a production-ready setup, and not a turnkey installer.

It is not designed to provide a comfortable user experience.

I welcome improvement suggestions and general discussion, but I don't guarantee support.

## How to use

1. Clone the repo.
2. Read this README first.
3. Read the README for each service first, before touching the specific files, begin with `bitcoind/README.md`.
4. Copy the files at the right places (see the respective service README).
5. Check if everything is correct before starting the service.

