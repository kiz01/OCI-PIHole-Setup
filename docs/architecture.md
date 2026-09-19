# System Architecture

## 1. Overview

This deployment is a self-hosted DNS filtering stack running on Oracle Cloud Infrastructure (OCI).

The architecture separates DNS filtering, recursive resolution, private network access, and infrastructure monitoring into distinct components.

```text
                         ┌──────────────────────┐
                         │      DNS Client      │
                         │  Laptop / Phone etc. │
                         └──────────┬───────────┘
                                    │
                                    │ DNS :53
                                    ▼
                         ┌──────────────────────┐
                         │       Pi-hole        │
                         │                      │
                         │ DNS filtering        │
                         │ Blocklists           │
                         │ DNS cache            │
                         │ Query logging        │
                         └──────────┬───────────┘
                                    │
                                    │ 127.0.0.1:5335
                                    ▼
                         ┌──────────────────────┐
                         │       Unbound        │
                         │                      │
                         │ Recursive resolver   │
                         │ DNSSEC validation    │
                         │ QNAME minimisation   │
                         │ DNS hardening        │
                         └──────────┬───────────┘
                                    │
                                    │ DNS resolution
                                    ▼
                         ┌──────────────────────┐
                         │ Authoritative DNS    │
                         │ Infrastructure       │
                         └──────────────────────┘
```

Administrative access uses a separate private connectivity path:

```text
                    ┌─────────────────────┐
                    │   Administrator     │
                    │   / Tailscale       │
                    └──────────┬──────────┘
                               │
                               │ Tailscale
                               ▼
                    ┌─────────────────────┐
                    │    OCI Server       │
                    │                     │
                    │ Ubuntu              │
                    │ Docker              │
                    │ Pi-hole             │
                    │ Unbound             │
                    │ Tailscale Exit Node │
                    └─────────┬───────────┘
                              │
                              │ Optional Internet egress
                              ▼
                    ┌─────────────────────┐
                    │ OCI Internet        │
                    │ Gateway             │
                    └─────────┬───────────┘
                              │
                              ▼
                           Internet
```

---

## 2. Infrastructure Layer

The server runs inside an Oracle Cloud Infrastructure Virtual Cloud Network.

The infrastructure consists of:

* OCI compute instance
* Virtual Cloud Network (VCN)
* Public subnet
* Internet Gateway
* Route table
* OCI network security controls

The server has a private address inside the OCI subnet and also has Tailscale connectivity for private remote access.

Sensitive infrastructure identifiers and addresses are intentionally excluded from this repository.

---

## 3. Operating System

The host operating system is Ubuntu Server.

The host provides:

* Linux networking
* SSH access
* Docker runtime
* Unbound service
* Tailscale networking
* System-level process and resource management

SSH administration is performed through the private Tailscale network rather than relying on direct public SSH access.

The host also provides IPv4 and IPv6 forwarding required for the Tailscale exit-node function.

The exit node is optional and can be enabled independently on authorized Tailscale clients.

---

## 4. Container Layer

Docker is used to run the application components.

The primary DNS filtering service runs as a Docker container managed through Docker Compose.

### Pi-hole container

Pi-hole uses:

```text
network_mode: host
```

With host networking, the container shares the host's network namespace.

As a result, Pi-hole can bind directly to host interfaces and ports rather than requiring Docker port publishing.

Persistent configuration is stored outside the container using bind mounts.

Conceptually:

```text
OCI Ubuntu Host
│
├── Docker
│   │
│   └── Pi-hole container
│       ├── /etc/pihole
│       └── /etc/dnsmasq.d
│
├── Unbound
│
└── Tailscale
```

---

## 5. DNS Layer

Pi-hole is the entry point for DNS requests.

Its responsibilities include:

1. Receiving DNS queries.
2. Checking configured blocklists and filtering rules.
3. Answering blocked queries locally.
4. Serving cached responses where available.
5. Forwarding permitted queries to Unbound.
6. Recording DNS query information according to the configured logging policy.

Pi-hole's configured upstream is the local Unbound instance:

```text
127.0.0.1:5335
```

This keeps the Pi-hole → Unbound communication local to the server.

---

## 6. Recursive Resolution

Unbound operates as the recursive DNS resolver behind Pi-hole.

The current deployment binds Unbound to:

```text
127.0.0.1:5335
```

It therefore does not serve DNS requests directly to remote network clients.

Unbound is responsible for recursive DNS resolution and DNSSEC validation.

The resulting flow is:

```text
Client
  │
  ▼
Pi-hole :53
  │
  ▼
Unbound 127.0.0.1:5335
  │
  ▼
Root / authoritative DNS infrastructure
```

---

## 7. Private Access and Exit-Node Layer

Tailscale provides private network connectivity to the server.

This is used for administrative access and allows trusted devices to reach services without requiring those services to be exposed directly to the public Internet.

The OCI server is also configured as an optional Tailscale exit node.

When a client selects the exit node, its general Internet traffic is routed through the OCI server and exits through the OCI Internet Gateway.

```text
Trusted network:

Client
  │
  │ Tailscale
  ▼
OCI Server
  │
  └── Private services

Exit node disabled


Public / untrusted network:

Client
  │
  │ Tailscale
  ▼
OCI Server
  │
  ├── DNS → Pi-hole → Unbound
  │
  └── Internet traffic
          │
          ▼
     OCI Internet Gateway
          │
          ▼
       Internet

Exit node enabled
```

The exit node is not required for normal Pi-hole operation. Clients can use Tailscale for private access while keeping their normal Internet connection.

The public documentation intentionally does not publish:

* Tailscale IP addresses
* Device names
* User/device identifiers
* Authentication material

---

## 8. Monitoring

Beszel is deployed as a separate Docker Compose stack.

It consists of:

* Beszel hub
* Beszel agent

The agent runs with host networking and collects host-level monitoring information.

The monitoring stack is considered a supporting component rather than part of the core DNS resolution path.

```text
                     ┌──────────────────┐
                     │    DNS Clients   │
                     └────────┬─────────┘
                              │
                              ▼
                       ┌─────────────┐
                       │   Pi-hole   │
                       └──────┬──────┘
                              │
                              ▼
                       ┌─────────────┐
                       │   Unbound   │
                       └─────────────┘


                       ┌─────────────┐
                       │   Beszel    │
                       │ Monitoring  │
                       └──────┬──────┘
                              │
                              ▼
                       Host / Docker
                       metrics
```

---

## 9. Trust Boundaries

The architecture contains several important trust boundaries.

### Client → Pi-hole

DNS clients are treated as network clients of the filtering service.

Access to DNS should be limited to intended networks.

### Pi-hole → Unbound

This is a local trust boundary.

Unbound only accepts DNS requests from the local host interface.

### Administrator → Server

Administrative access is provided through Tailscale.

SSH credentials are not stored in this repository.

### Client → OCI Exit Node

When the Tailscale exit node is enabled, the client becomes dependent on the OCI server for general Internet egress.

This introduces an additional forwarding and network trust boundary.

Exit-node forwarding should therefore be treated separately from the DNS filtering function.

### OCI Server → Internet

Exit-node traffic leaves the infrastructure through the OCI Internet Gateway.

The OCI network configuration remains responsible for controlling the server's externally reachable services; exit-node forwarding does not replace those controls.

### Internet → Server

Public network access is controlled at the OCI networking layer and must be reviewed independently of Pi-hole configuration.

Pi-hole itself should not be treated as a host firewall.

---

## 10. Security Principle

The deployment follows a layered security model:

```text
┌───────────────────────────────────────────┐
│ OCI network security                      │
├───────────────────────────────────────────┤
│ Tailscale private connectivity            │
├───────────────────────────────────────────┤
│ Optional Tailscale exit-node forwarding   │
├───────────────────────────────────────────┤
│ Ubuntu host                               │
├───────────────────────────────────────────┤
│ Docker isolation                          │
├───────────────────────────────────────────┤
│ Pi-hole DNS filtering                     │
├───────────────────────────────────────────┤
│ Unbound recursive DNS + DNSSEC validation │
└───────────────────────────────────────────┘
```

No single component is intended to provide all security controls.

In particular, DNS filtering and firewalling are separate responsibilities.

The exit-node function is an optional network capability and is not required for DNS filtering or private service access.

---

## 11. Current Deployment Scope

This document describes the current working architecture.

It does not include future planned components such as:

* Parental-control policy management
* Custom VPN infrastructure
* Android agents
* Automated security alerting
* SIEM integration
* Advanced traffic inspection

Those components may be documented separately if they are implemented in the future.
