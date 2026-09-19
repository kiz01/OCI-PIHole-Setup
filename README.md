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
* **Tailscale** — private network access and optional Internet exit-node routing
* **Beszel** — optional host/container monitoring

The goal of this repository is to document the deployment, architecture, networking, DNS flow, security considerations, hardening decisions, and manual rebuild process of the running system.

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

When the Tailscale exit node is enabled on a client, Internet traffic can additionally follow:

```text
Client
  │
  │ Tailscale
  ▼
OCI Ubuntu server
  │
  │ Internet egress
  ▼
OCI Internet Gateway
  │
  ▼
Internet
```

The Pi-hole container uses **host networking**, allowing Pi-hole's DNS and web services to bind directly to host interfaces.

Unbound is deliberately bound to the local host interface and is used as Pi-hole's upstream resolver.

The Tailscale exit node is an **optional capability**. Clients can use Tailscale for private access without routing their general Internet traffic through OCI.

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

The current deployment does not require the Docker `NET_ADMIN` capability.

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

The OCI server is also configured as an **optional Tailscale exit node**, allowing authorized Tailscale clients to route their general Internet traffic through the OCI server.

This provides two usage modes:

```text
Trusted / personal network
Client → Tailscale → private services
                    Exit node OFF

Public / untrusted network
Client → Tailscale → OCI exit node → Internet
                         │
                         └── Pi-hole / Unbound DNS
```

Private Tailscale addresses and device-specific information are intentionally excluded from this repository.

### Docker

Pi-hole runs as a Docker container with persistent configuration stored outside the container.

The deployment uses Docker Compose.

Persistent Pi-hole data includes configuration and database files under the deployment directory.

### Beszel

Beszel provides optional host and container monitoring.

The current deployment keeps the Beszel agent without access to the Docker socket.

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
        ├── Optional Tailscale exit node
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
* Treating exit-node forwarding as an additional network trust boundary

Pi-hole's DNS listener is configured to listen broadly at the application level, so **network-level access controls are an important part of the security boundary**.

Additional hardening includes:

* Pi-hole running without `NET_ADMIN`
* Pi-hole's built-in NTP listener disabled
* Unbound restricted to localhost
* Beszel agent running without Docker socket access
* IPv4 and IPv6 forwarding enabled for Tailscale exit-node operation
* Runtime state and credentials excluded from version control

The exit-node capability has been verified with multiple Tailscale clients, including Internet egress through the OCI public IP and continued Pi-hole DNS filtering.

See [`docs/security.md`](docs/security.md) for the detailed security model and [`docs/hardening-log.md`](docs/hardening-log.md) for the documented hardening history.

---

## Repository Structure

```text
.
├── README.md
├── LICENSE
├── .gitignore
├── docs/
│   ├── architecture.md
│   ├── networking.md
│   ├── dns-stack.md
│   ├── security.md
│   ├── hardening-log.md
│   └── rebuild.md
└── examples/
    ├── pihole/
    │   └── compose.yaml
    ├── unbound/
    │   └── pi-hole.conf
    └── beszel/
        └── compose.yaml
```

---

## Documentation

| Document | Description |
| --- | --- |
| [`architecture.md`](docs/architecture.md) | Overall system architecture and component relationships |
| [`networking.md`](docs/networking.md) | OCI, Ubuntu, Docker, Tailscale, and exit-node networking |
| [`dns-stack.md`](docs/dns-stack.md) | Pi-hole → Unbound DNS resolution flow |
| [`security.md`](docs/security.md) | Security boundaries, risks, and hardening considerations |
| [`hardening-log.md`](docs/hardening-log.md) | Chronological record of hardening changes and verification |
| [`rebuild.md`](docs/rebuild.md) | Manual rebuild procedure and post-rebuild verification |

---

## Configuration Examples

The [`examples/`](examples/) directory contains sanitized reference configurations for the main services.

These examples:

* Use placeholders for secrets and deployment-specific values
* Do not contain real infrastructure addresses or credentials
* Reflect the security-relevant configuration of the current deployment
* Are intended as reference material rather than automated deployment files

Before rebuilding the system, review the current official documentation for the relevant software and adapt these examples to the versions being deployed.

---

## Rebuild Approach

This project intentionally does **not** provide an automated installation or deployment script.

The rebuild process is documented as a manual procedure in [`docs/rebuild.md`](docs/rebuild.md).

This approach is intentional: infrastructure software, package versions, installation procedures, and cloud-provider interfaces can change over time. A manually verified rebuild is preferred over blindly following a potentially outdated automation script.

The documentation and sanitized configuration examples serve as reference material. A future rebuild should always verify the current upstream installation and configuration requirements before proceeding.

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

Runtime state such as Pi-hole databases, query logs, DHCP leases, TLS private material, Docker application data, Tailscale state, and other host-specific runtime data should remain on the server and outside version control.

---

## Current Scope

This repository currently focuses on documenting the existing Pi-hole deployment and its supporting cloud/network infrastructure.

It does **not** currently attempt to provide:

* Parental-control policies
* Custom VPN infrastructure
* Android device agents
* Automated alerting
* Centralized SIEM functionality
* Advanced traffic inspection
* Automated deployment scripts

The Tailscale exit node is documented as an optional network capability rather than a requirement for normal Pi-hole operation.

Additional capabilities may be considered separately in the future.

---

## License

This project is licensed under the MIT License.

See [`LICENSE`](LICENSE).
