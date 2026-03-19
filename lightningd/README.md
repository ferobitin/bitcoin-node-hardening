# lightningd Service (Core Lightning)

Tested with:

- Core Lightning v25.12.1
- Bitcoin Core v30.2.0
- Debian 13 "Trixie"
- systemd 257

## Overview

Core Lightning (`lightningd`) is a lightning node implementation that operates on top of `bitcoind`.  
It is a tool to manage lightning channels, HTLCs and routing payment paths.

`lightningd` depends on `bitcoind` for validated blockchain data via JSON-RPC,  
runs as a hardened service and is managed via systemd.

## Design

### User & Permissions

`lightningd` runs as a dedicated lightning user with no interactive login shell and minimal permissions.

### Paths

The datadir ("lightning-dir") is located at `/srv/lightning`  
and the configuration file at `/etc/lightning/config`.

Runtime logs are written to `/var/log/lightning/`.

Executable binaries and the system integration are stored at the system layer,  
while critical and general data are isolated in a dedicated lightning-dir.

### Dependency on Bitcoin Core

`lightningd` does not work without `bitcoind`. It needs a full bitcoin node for blockchain validation,  
channel transaction monitoring, broadcasting of commitment and HTLC transactions and mempool information.

`lightningd` communicates with `bitcoind` via a local-only JSON-RPC interface.  
The startup depends on `bitcoind` via the `After=` and `Requires=` systemd directives.

### Connectivity

Clearnet outbound connections are disabled. 
Outbound Lightning peer connections are routed through a local Tor SOCKS5 proxy. 
Inbound connectivity is available via an external managed hidden service.

See `/tor/README.md`.

## Systemd Hardening

Because the service manages lightning channels and funds, the unit is hardened to minimize  
the impact of potential attacks, bugs and corruption.

### Filesystem Isolation

`ProtectSystem=strict` mounts the root filesystem read-only,  
`ProtectHome=true` makes `/root` and `/run/user` inaccessible.

`PrivateTmp` creates private `/tmp` and `/var/tmp`, preventing sharing with other services.

Only explicitly needed paths are made accessible via `ReadWritePaths`.

`/dev`, `/proc` and `/sys` are protected via `PrivateDevices=true`, `ProtectKernelTunables=true`  
and `ProtectControlGroups=true`.

### Address Families

Since the node routes its traffic via the local Tor SOCKS5 proxy at `127.0.0.1:9052`,  
only `AF_INET` is needed for network communication.

`AF_UNIX` is required for local IPC.

`RestrictAddressFamilies=` creates an allow-list.

The complete file can be found in `/lightningd/lightningd.service`.

## Configuration (config)

`lightningd` is configured via `/etc/lightning/config`.

### General

`network=bitcoin` sets the network within which `lightningd` operates.

Alternatively, you can choose:

- `testnet`
- `signet`
- `regtest`

for risk-free testing environments.

### Bitcoin Core Backend

`lightningd` has full control over `bitcoind` via the local JSON-RPC interface  
(see `/bitcoind/README.md`) in order to access the provided data  
(see *Dependency on Bitcoin Core* above).

To make `.cookie` authentication work, we grant access to the bitcoin datadir via  
`bitcoin-datadir=/srv/bitcoin`, where the cookie file is stored.

`bitcoin-rpcport=8332` and `bitcoin-rpcconnect=127.0.0.1` tell `lightningd` where `bitcoind` is listening.

### Networking (Tor)

`always-use-proxy=true` enforces that all outbound is routed through a local Tor SOCKS5 proxy (`proxy=127.0.0.1`).

`bind-addr=127.0.0.1:9735` binds the lightning TCP listener to localhost, allowing inbound connections only through the Tor hidden service (see `/tor/README.md`).

Direct clearnet connections are effectively disabled.

See `/tor/README.md` for further details.

The complete file can be found in `/lightningd/config`.

## Trust & Security

### Funds & Custody

Lightning normally operates as a hot custody system.  
Key material is stored in the lightning-dir (`hsm_secret`).

`hsm_secret` controls the node identity, channel keys and ultimately access to all on-chain and lightning funds.

If you lose the file, you lose access to your funds.  
If an attacker obtains access to it, the attacker can control your node and its funds.

If the node is compromised, your funds are at risk.

The lightning-dir should therefore be backed up regularly, especially the `hsm_secret` file.

### Bitcoin Core RPC

See `/bitcoind/README.md` for details about the RPC configuration and security considerations.

### Host Integrity Assumption

The entire security architecture becomes obsolete if an attacker gains root access,  
physical access to the machine, or if the kernel is compromised.

## Operational Notes

### Shutdown

Avoid killing the process directly or shutting down the host while the service is running.  
This may interrupt channel state updates.

### Logs

Logging is handled via `journald`:

`journalctl -u lightningd`

### Backups

The most critical file is `/srv/lightning/hsm_secret` (see *Funds & Custody*).

Regular backups of the lightning-dir are recommended and should contain at least:

- `hsm_secret`
- `lightningd.sqlite3`
- optionally the `config`
