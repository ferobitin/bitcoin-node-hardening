# bitcoind Service

Tested with:
- Bitcoin Core v30.2.0
- Debian 13 "Trixie"
- systemd 257

## Overview

This node primarily acts as a Tor-only backend for Core Lightning.

Core Lightning (`lightningd`) depends on `bitcoind` for validated blockchain data via JSON-RPC.  
`bitcoind` is managed via systemd.

## Design

### User & Permissions

`bitcoind` runs as a dedicated `bitcoin` system user with no interactive login shell and minimal permissions.

### Paths

The datadir is located at `/srv/bitcoin`,  
the configuration file at `/etc/bitcoin/bitcoin.conf`  
and the runtime state is stored at `/var/lib/bitcoin`.

Executable binaries and the system integration are stored at the system layer,  
while blockchain state and wallet data are isolated in a dedicated datadir.

### RPC Access

RPC is bound to localhost and authenticated via the cookie file (by default), and used by Core Lightning and `bitcoin-cli`.

### Connectivity

Outbound peer connections are routed through a local Tor SOCKS5 proxy.  
Clearnet outbound connections are disabled.  
Inbound connectivity is available only via onion service.

### Boot Order

The startup ordering is enforced via `After=` and `Wants=`.

`bitcoind.service` starts after `tor.service`.  
`lightningd.service` starts after `bitcoind.service`.

## Systemd Hardening

The unit is designed to be as restrictive as possible, without breaking the service.

### Filesystem Isolation

`UMask=0077` restrains default permissions for newly created files and directories.
Group access is only permitted where explicitly required (see RPC Access Boundary).

The filesystem is mounted read-only via `ProtectSystem=strict`.

`PrivateTmp` creates private `/tmp` and `/var/tmp`, preventing sharing with other services.

`ReadWritePaths=` explicitly gives rw permissions for the datadir and the state directory.

HWI uses the `StateDirectory` as a supplementary Home for HWI configuration:  
since `ProtectHome=true` blocks access to `/home`, `XDG_CONFIG_HOME` is redirected to `/var/lib/bitcoin/config`.

`/proc` and `/sys` are protected via `ProtectKernelTunables=true` and `ProtectControlGroups=true`.


### Devices

Device access is denied by default.
The only exception is granted for USB character devices, in order to enable communication between HWI and hardware wallets.

### Address Family Restrictions

`AF_INET` is needed for local RPC communication (`bitcoin-cli`, `lightningd`) and TCP connection (Tor SOCKS5 and Tor Control Port).

`AF_UNIX` is required for local IPC.

`AF_NETLINK` is enabled for hardware wallet enumeration via HWI.


The complete file can be found in `/bitcoind/bitcoind.service`.

## Configuration (`bitcoin.conf`)

In this setup, the config is minimal and security-focused:

Bitcoin Core runs Tor-only and local RPC only, provides blockchain data and indexes for Core Lightning and wallet workflow.

### RPC

`server=1` enables the JSON-RPC server, required by `bitcoin-cli` and `lightningd`.

Cookie authentication is the default, avoiding static plaintext passwords.
The cookie file is stored in the datadir (`rpccookieperms=group` allows group access).

`rpcbind=127.0.0.1` binds RPC to loopback only.
`rpcallowip=127.0.0.1` is practically redundant but it's a nice additional defense layer.

### Network

The setup runs Tor only.

`onlynet=onion` restricts peer connection to the onion network.

Outbound P2P traffic is routed via a local Tor SOCKS5 proxy:
`proxy=127.0.0.1:9050` (see `/tor/README.md`).

`listen=1` is default to allow general inbound connectivity.

`listenonion=1` is set in order to enable onion-inbound.
`torcontrol=127.0.0.1:9051` allows `bitcoind` to set up an onion service (via the Tor control port).

Since clearnet peer addresses are not used:
`dnsseed=0` avoids unnecessary DNS seed lookups via Tor.

`discover=0` is set explicitly to enforce and document the Tor-only network boundary,  
(although it would be disabled by `proxy=` anyway).

### Indexes (optional)

`txindex=1` may be required by certain CLN plugins, enables historical transaction lookup.
(Increases disk usage and IBD time significantly)

`blockfilterindex=1` creates BIP158 compact block filters (nice to have; wallet rescans are faster).
(Slightly increases disk usage and IBD time)

`coinstatsindex=1` might be useful for monitoring and research use cases.
(Trades additional disk space and IBD time)

### Resources & Performance (optional)

`dbcache=4000` allocates ~4 GB RAM for the UTXO set cache (as well as block index and coin database buffers).

`maxconnections=40` limits the number of concurrent peers to 40 in order to reduce potential resource pressure on Tor.

Adjust this to your individual needs and hardware.

### External Signer

`signer=/usr/local/bin/hwi` sets the external signer interface (HWI) to interact with hardware wallets.

See `/environment/README.md`.


The complete file can be found in `/bitcoind/bitcoin.conf`.

## Trust & Security

### RPC Access Boundary

RPC access is only granted to the `bitcoin` user and members of the `bitcoin` group, enforced through Unix permissions on the RPC cookie file.

`UMask=0077` (see Systemd Hardening) is crucial to restrict the default permissions on all other files created by `bitcoind`, since the cookie file lives in the datadir.
This way, wallets and other sensitive elements remain inaccessible to group members and others.

RPC is bound to loopback only (see "RPC").

The cookie is the only way to gain RPC control over `bitcoind`, and it grants full access.

### Private Keys

Private keys should not reside on the node.
Use a watch-only wallet and sign PSBTs with a hardware wallet via HWI.

### Network Exposure

This node is not reachable over clearnet.
P2P runs Tor-only and RPC is local-only.

See "Connectivity" and "Network".

### Host Integrity Assumption

The entire security architecture becomes obsolete if an attacker gains root access, physical access to the machine, or if the kernel is compromised.

## Operational Notes

### Shutdown

Stop `bitcoind` before shutdown via:

`systemctl stop bitcoind`

Avoid using `kill -9` or shutting down the system while bitcoind is running, to prevent potential data corruption.

### Logs

Logs can be inspected via:

- `journalctl -u bitcoind`
- `tail -f /srv/bitcoin/debug.log`

### Backups

Blockchain and chainstate data does not need to be archived.

Focus on backing up:

- wallet metadata
- wallet files and directories
- `settings.json`
- `onion_v3_private_key`
- `bitcoin.conf`
- `bitcoind.service`
