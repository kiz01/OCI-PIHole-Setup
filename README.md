# OCI Pi-hole Setup

Self-hosted Pi-hole DNS filtering on Oracle Cloud Infrastructure (OCI), with Tailscale for private access and Unbound for recursive DNS resolution.

## Overview

This project documents a working self-hosted DNS filtering server deployed on Oracle Cloud Infrastructure.

The system combines:

* **Oracle Cloud Infrastructure (OCI)** — cloud compute and network infrastructure
* **Ubuntu Server** — host operating system
* **Docker** — container runtime
* **Pi-hole** — DNS filtering, blocking, caching, and query visibility
* **Unbound** — local recursive DNS resolver with DNSSEC validation
* **Tailscale** — private network access to the server
* **Beszel** — optional host/container monitoring

The goal of this repository is to document the deployment, architecture, networking, DNS flow, and security considerations of the running system.

> **Status:** This repository documents the current working deployment. It is not intended to represent every possible Pi-hole or OCI configuration.

---

## Architecture

At a high level, DNS requests follow this path:

```text
Client
  │
  │ DNS query
  ▼
Pi-hole
  │
  │ blocked?
  ├──────────────► Block response
  │
  │ allowed
  ▼
Unbound
  │
  │ Recursive DNS resolution
  │ DNSSEC validation
  ▼
Authoritative DNS infrastructure
  │
  ▼
Internet
```

Administrative access follows a separate path:

```text
Administrator
     │
     ▼
 Tailscale network
     │
     ▼
 OCI Ubuntu server
     │
     ├── Pi-hole
     ├── Unbound
     └── Docker services
```

The Pi-hole container uses **host networking**, allowing Pi-hole's DNS and web services to bind directly to host interfaces.

Unbound is deliberately bound to the local host interface and is used as Pi-hole's upstream resolver.

---

## Core Components

### Pi-hole

Pi-hole provides:

* DNS-based filtering
* Blocklists
* DNS caching
* Query logging
* Local DNS functionality
* Protection against several common DNS-bypass mechanisms

The current deployment uses Pi-hole 6.x.

Pi-hole receives client DNS requests on port `53` and forwards permitted queries to the local Unbound resolver.

### Unbound

Unbound provides recursive DNS resolution for Pi-hole.

The current configuration:

* Listens on `127.0.0.1:5335`
* Accepts both UDP and TCP DNS queries
* Uses DNS root hints
* Enables DNSSEC trust-anchor configuration
* Enables QNAME minimisation
* Enables DNS hardening options
* Does not accept remote clients

Pi-hole therefore does not depend directly on a public third-party DNS resolver for normal upstream resolution.

### Tailscale

Tailscale provides private network connectivity for administrative access and DNS clients.

The server is accessed through its Tailscale address rather than relying on SSH access through the public OCI address.

Private Tailscale addresses and device-specific information are intentionally excluded from this repository.

### Docker

Pi-hole runs as a Docker container with persistent configuration stored outside the container.

The deployment uses Docker Compose.

Persistent Pi-hole data includes configuration and database files under the deployment directory.

---

## OCI Network

The server is deployed inside an OCI Virtual Cloud Network (VCN) with a public subnet and Internet Gateway route.

The public repository intentionally does **not** contain:

* Public IP addresses
* Private IP addresses
* Tailscale addresses
* Device names
* SSH credentials
* API keys
* Passwords
* Private certificates or keys

The exact OCI network configuration should be adapted to the deployment environment.

---

## Security Model

Security is provided through multiple layers rather than relying on Pi-hole itself as a firewall.

The deployment separates the responsibilities of:

```text
OCI network controls
        │
        ▼
Ubuntu host
        │
        ├── Tailscale private access
        │
        ├── Docker
        │
        ├── Pi-hole
        │
        └── Unbound
```

Important considerations include:

* Restricting access to administrative services
* Preventing unintended public DNS exposure
* Keeping Unbound restricted to localhost
* Protecting Pi-hole credentials
* Never committing private keys or secrets
* Keeping generated databases and runtime files out of version control
* Reviewing OCI ingress rules before exposing additional services

Pi-hole's DNS listener is configured to listen broadly at the application level, so **network-level access controls are an important part of the security boundary**.

See [`docs/security.md`](docs/security.md) for the detailed security model.

---

## Repository Structure

```text
.
├── README.md
├── LICENSE
├── .gitignore
└── docs/
    ├── architecture.md
    ├── networking.md
    ├── dns-stack.md
    └── security.md
```

---

## Documentation

| Document                                  | Description                                              |
| ----------------------------------------- | -------------------------------------------------------- |
| [`architecture.md`](docs/architecture.md) | Overall system architecture and component relationships  |
| [`networking.md`](docs/networking.md)     | OCI, Ubuntu, Docker, and Tailscale networking            |
| [`dns-stack.md`](docs/dns-stack.md)       | Pi-hole → Unbound DNS resolution flow                    |
| [`security.md`](docs/security.md)         | Security boundaries, risks, and hardening considerations |

---

## Configuration Safety

This repository is intended to remain safe for public GitHub publication.

The following should **never** be committed:

```text
.env
.env.*
*.key
*.pem
*.crt
credentials
password files
SSH private keys
API tokens
Tailscale authentication secrets
Pi-hole databases
runtime logs
DHCP lease files
generated cache files
```

Configuration examples should use placeholders instead of real credentials or infrastructure identifiers.

---

## Current Scope

This repository currently focuses on documenting the existing Pi-hole deployment.

It does **not** currently attempt to provide:

* Parental-control policies
* Custom VPN infrastructure
* Android device agents
* Automated alerting
* Centralized SIEM functionality
* Advanced traffic inspection
* Automated deployment scripts

Those may be considered separately in the future.

---

## License

This project is licensed under the MIT License.

See [`LICENSE`](LICENSE).
