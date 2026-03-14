# Tor Service

Tested with:

- Tor version 0.4.8.16
- Debian 13 "Trixie"
- systemd 257

## Overview

Tor provides network privacy for `lightningd` and `bitcoind`.

## Design

Tor runs as a separate systemd service and routes all node network traffic.

```
                        Tor Network
                             ▲
                             │ outbound
                             │
                             │ inbound
                             ▼
                            Tor
                       (tor daemon)
                       ▲         │
         outbound via  │         │ inbound via onion (hidden) services
         SOCKS proxies │         │
                       │         │
         ┌─────────────┘         └─────────────┐
         │                                     │
         │                                     ▼

  127.0.0.1:9050                         bitcoin.onion:8333
  → bitcoind outbound                    127.0.0.1:8333
                                         → bitcoind inbound

  127.0.0.1:9052                         lightning.onion:9735
  → lightningd outbound                  127.0.0.1:9735
                                         → lightningd inbound
```
Tor is installed from the Debian package (`apt`) and uses the default systemd unit. 
No modifications to the unit file were made, therefore it is not included in this repository.

## Paths

Configuration: `/etc/tor/torrc`  
State and keys: `/var/lib/tor/*`

## Configuration

### Authentication

Tor uses cookie authentication (`CookieAuthentication 1`) with group-readable cookies (`CookieAuthFileGroupReadable 1`) to allow controlled access to the control port.

### SOCKS Proxies

Local SOCKS5 proxies are exposed at:

- `SocksPort 127.0.0.1:9050` for `bitcoind`
- `SocksPort 127.0.0.1:9052` for `lightningd`

Different SOCKS ports are used for different services to isolate traffic.

### Hidden Services (Inbound)

Mapping from external onion addresses to local ports.

Tor publishes node services as onion addresses and forwards incoming connections to the configured port:

`HiddenServiceDir /var/lib/tor/lightning-service/`  
`HiddenServicePort 9735 127.0.0.1:9735`

```
Tor network → lightning.onion:9735 → Tor → 127.0.0.1:9735 → lightningd
```

In this setup, `bitcoind` manages its onion service automatically via the Tor control port:

`ControlPort 127.0.0.1:9051`

The complete configuration file can be found in `/tor/torrc`.

## Backups

The private key determines the `.onion` address. If you lose these keys, Tor will generate a new key pair
and the onion address will change. Changing the onion address my break peer connections and known service
endpoints.

These keys should therefore be backed up.

Normally the keys are stored under `/var/lib/tor/*`; `bitcoind` stores the keys in the datadir.
