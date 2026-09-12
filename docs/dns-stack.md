# DNS Stack

## 1. Overview

The DNS stack consists of two primary components:

1. **Pi-hole** — DNS filtering and local DNS service
2. **Unbound** — recursive DNS resolver

The two services operate together rather than independently.

```text
DNS Client
    │
    │ TCP/UDP 53
    ▼
┌───────────────┐
│    Pi-hole    │
│               │
│ Filtering     │
│ Blocklists    │
│ Cache         │
│ Query logging │
└───────┬───────┘
        │
        │ 127.0.0.1:5335
        ▼
┌───────────────┐
│    Unbound    │
│               │
│ Recursive DNS │
│ DNSSEC        │
│ Validation    │
│ Hardening     │
└───────┬───────┘
        │
        ▼
Authoritative DNS infrastructure
```

---

## 2. Pi-hole

Pi-hole is the primary DNS endpoint for clients.

The current deployment uses Pi-hole 6.x in a Docker container.

Pi-hole provides:

* DNS query handling
* Domain blocking
* Blocklists
* DNS caching
* Query logging
* Local DNS functionality
* DNS-bypass mitigation features

The primary DNS service listens on:

```text
TCP/UDP 53
```

---

## 3. Pi-hole Upstream

Pi-hole forwards permitted queries to the local Unbound instance:

```text
127.0.0.1#5335
```

This is configured as the Pi-hole upstream resolver.

Therefore, the normal resolution path is:

```text
Client
  │
  ▼
Pi-hole
  │
  ▼
127.0.0.1:5335
  │
  ▼
Unbound
```

Pi-hole does not directly use a public DNS provider as its configured upstream in this deployment.

---

## 4. Query Processing

A simplified Pi-hole decision process is:

```text
                    DNS Query
                        │
                        ▼
                 ┌─────────────┐
                 │   Pi-hole   │
                 └──────┬──────┘
                        │
                 Check filtering
                        │
              ┌─────────┴─────────┐
              │                   │
            Block                Allow
              │                   │
              ▼                   ▼
       Block response          Unbound
                                  │
                                  ▼
                              DNS result
```

Blocked requests are answered locally.

Permitted requests continue to Unbound for recursive resolution.

---

## 5. Blocking

DNS filtering is active in the current configuration.

Pi-hole uses `NULL` mode for blocked responses.

In this mode, blocked DNS queries are answered with unspecified addresses rather than being forwarded upstream.

The deployment also uses EDNS information for blocked responses.

---

## 6. Blocklists

The current deployment includes a hosts-based blocklist.

The configured list is:

```text
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
```

Additional lists can be added through Pi-hole's configuration if required.

Blocklists are an application-level filtering mechanism.

They should not be confused with:

* Firewall rules
* Network ACLs
* Intrusion prevention
* Web filtering
* Endpoint security

---

## 7. DNS Cache

Pi-hole maintains a DNS cache.

The current configuration specifies a cache size of:

```text
10000
```

An additional cache optimization period is configured.

Caching can reduce repeated recursive lookups and improve response times for frequently requested domains.

Unbound also maintains its own recursive DNS caches.

This results in caching at two different layers:

```text
Client
  │
  ▼
Pi-hole cache
  │
  │ cache miss
  ▼
Unbound cache
  │
  │ cache miss
  ▼
Recursive resolution
```

---

## 8. Unbound

Unbound is the recursive resolver used by Pi-hole.

Unlike a conventional forwarding configuration, Unbound performs recursive DNS resolution itself.

Its role is therefore different from Pi-hole:

| Component | Primary responsibility                |
| --------- | ------------------------------------- |
| Pi-hole   | Filtering and DNS service for clients |
| Unbound   | Recursive DNS resolution              |

Keeping these responsibilities separate makes the architecture easier to reason about and troubleshoot.

---

## 9. Unbound Listener

Unbound listens only on the local interface:

```text
127.0.0.1:5335
```

Both DNS transport protocols are enabled:

```text
UDP
TCP
```

Remote clients are not intended to query Unbound directly.

The Unbound configuration also explicitly refuses non-local IPv4 clients.

Conceptually:

```text
Remote client
     │
     X
     │
     └── Unbound is not directly exposed

Pi-hole
     │
     ▼
127.0.0.1:5335
     │
     ▼
Unbound
```

---

## 10. Recursive DNS

When Unbound does not have a valid cached response, it performs recursive resolution through the DNS hierarchy.

Conceptually:

```text
                 Unbound
                    │
                    ▼
                Root DNS
                    │
                    ▼
              TLD servers
                    │
                    ▼
          Authoritative server
                    │
                    ▼
               DNS answer
```

This allows Unbound to resolve domains without depending on a separate recursive DNS provider.

---

## 11. DNSSEC

DNSSEC validation is handled by Unbound.

The configuration includes an automatic trust-anchor file:

```text
/var/lib/unbound/root.key
```

Pi-hole itself does not perform the final DNSSEC validation in this architecture.

This distinction is important.

The configuration can therefore be represented as:

```text
Client
  │
  ▼
Pi-hole
  │
  │ filtering
  ▼
Unbound
  │
  │ recursive resolution
  │ DNSSEC validation
  ▼
DNS infrastructure
```

Pi-hole's own DNSSEC setting should therefore not be interpreted in isolation when evaluating the security properties of the complete DNS stack.

---

## 12. DNS Hardening

The Unbound configuration enables several hardening features, including:

* Glue hardening
* DNSSEC-stripping hardening
* QNAME minimisation
* DNSSEC key prefetching
* Response prefetching
* EDNS buffer sizing

These settings are intended to improve DNS privacy, integrity, resilience, or interoperability.

The exact values are maintained in the live server configuration rather than being reproduced here as a production configuration file.

---

## 13. IPv4 / IPv6

The current Unbound configuration enables IPv4 DNS operation and disables IPv6 operation:

```text
do-ip4: yes
do-ip6: no
```

Pi-hole itself may have IPv6-related listener capabilities depending on the host and runtime configuration.

Therefore, IPv6 exposure should be evaluated separately from Unbound's recursive resolution configuration.

---

## 14. Local DNS

Pi-hole is configured with the local DNS domain:

```text
lan
```

The domain is treated as local.

Pi-hole does not forward queries for this local domain upstream unless an appropriate reverse-server configuration is present.

No reverse server configuration is currently defined.

---

## 15. DNS-Bypass Mitigation

Several Pi-hole special-domain protections are enabled.

These include mechanisms intended to reduce DNS bypass through:

* Firefox automatic DNS-over-HTTPS detection
* Apple iCloud Private Relay
* Discovery of Designated Resolvers

These controls are useful as part of the DNS policy layer.

They are not equivalent to endpoint-level enforcement.

A device or application capable of using an alternative encrypted DNS mechanism may still require additional controls outside Pi-hole.

---

## 16. Query Logging

DNS query logging is enabled in Pi-hole.

This provides visibility into DNS activity and supports:

* Troubleshooting
* Blocklist verification
* Client/domain analysis
* Operational monitoring

Query logs can contain sensitive information.

For that reason, runtime databases and logs from the live server are **not** included in the public repository.

---

## 17. Security Boundary

The most important DNS security boundary is the separation between:

```text
Client-facing DNS
        │
        ▼
     Pi-hole
        │
        ▼
Local recursive resolver
        │
        ▼
     Unbound
```

Pi-hole is responsible for filtering.

Unbound is responsible for recursive resolution and DNSSEC validation.

Network controls are responsible for deciding who can reach the client-facing DNS service.

These responsibilities should not be conflated.

---

## 18. Open Resolver Consideration

Pi-hole's current listener configuration allows broad interface binding.

This is intentional in the current deployment but introduces an important security requirement:

> The DNS service must not be reachable by unauthorized external clients.

If port `53` were exposed to the public Internet without appropriate network restrictions, the server could potentially function as an open recursive DNS service through the Pi-hole → Unbound chain.

Therefore, OCI network controls and any other applicable access-control layers are critical to the security of this deployment.

---

## 19. Troubleshooting Flow

DNS problems can be isolated by testing each layer separately.

### Test Pi-hole

```bash
dig @<PIHOLE_ADDRESS> example.com
```

### Test Unbound locally

```bash
dig @127.0.0.1 -p 5335 example.com
```

### Check listeners

```bash
sudo ss -tulpn | grep -E ':(53|5335)\b'
```

### Check Unbound

```bash
sudo systemctl status unbound
```

### Check Pi-hole

```bash
docker ps
```

The goal is to identify whether a failure occurs at:

```text
Client
  ↓
Network access
  ↓
Pi-hole
  ↓
Unbound
  ↓
Recursive DNS
```

---

## 20. Summary

The current DNS architecture deliberately separates filtering from recursive resolution:

```text
                 ┌──────────────────┐
                 │      Client      │
                 └────────┬─────────┘
                          │
                          │ DNS
                          ▼
                 ┌──────────────────┐
                 │     Pi-hole      │
                 │                  │
                 │ Filtering        │
                 │ Blocklists       │
                 │ Caching          │
                 │ Query logging    │
                 └────────┬─────────┘
                          │
                          │ localhost:5335
                          ▼
                 ┌──────────────────┐
                 │     Unbound      │
                 │                  │
                 │ Recursive DNS    │
                 │ DNSSEC validation│
                 │ DNS hardening    │
                 └────────┬─────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │ DNS hierarchy    │
                 └──────────────────┘
```

This provides a self-hosted DNS filtering and recursive resolution stack while keeping the recursive resolver restricted to the local host.
