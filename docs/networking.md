# Networking

## 1. Overview

The Pi-hole server runs inside an Oracle Cloud Infrastructure (OCI) Virtual Cloud Network (VCN).

Networking is divided into several layers:

```text
Internet
   │
   ▼
OCI Internet Gateway
   │
   ▼
OCI VCN
   │
   ▼
Public Subnet
   │
   ▼
Ubuntu Server
   │
   ├── ens3
   ├── Tailscale
   ├── Docker networking
   ├── Pi-hole
   └── Unbound
```

The public repository intentionally omits public IP addresses, private host addresses, Tailscale addresses, and device-specific identifiers.

---

## 2. OCI Network

The deployment uses:

* OCI Virtual Cloud Network
* Public subnet
* Regional subnet configuration
* Internet Gateway
* Route table
* OCI Security List / network security controls

### VCN

The VCN uses a private IPv4 address space.

The server resides in a dedicated subnet within the VCN.

Infrastructure identifiers such as the VCN name, compartment names, and exact addressing are intentionally not published.

---

## 3. Subnet

The server is deployed in a public OCI subnet.

A public subnet does not automatically mean that every service on the instance is reachable from the Internet.

Actual reachability depends on the combined effect of:

* OCI route configuration
* OCI Security Lists
* Network Security Groups, if attached
* Host firewall rules
* Service listener configuration
* Docker networking and forwarding rules

Therefore, the presence of an Internet Gateway should not by itself be interpreted as unrestricted inbound access.

---

## 4. Routing

The subnet has a default route toward the OCI Internet Gateway.

Conceptually:

```text
0.0.0.0/0
     │
     ▼
Internet Gateway
     │
     ▼
OCI VCN
     │
     ▼
Server subnet
```

The server uses the OCI subnet gateway for normal Internet-bound traffic.

The routing configuration is separate from inbound access control.

---

## 5. Host Network Interfaces

The Ubuntu host has several network interfaces serving different purposes.

### Primary OCI interface

The primary network interface connects the server to the OCI subnet.

Conceptually:

```text
ens3
  │
  └── OCI private network
```

The interface carries normal host traffic and provides the default route toward the OCI network gateway.

### Tailscale interface

Tailscale creates a virtual interface for the private overlay network.

Conceptually:

```text
tailscale0
    │
    └── Private Tailscale network
```

Administrative SSH access is performed through the Tailscale network.

The actual Tailscale addresses and device identifiers are intentionally excluded from this repository.

### Docker interfaces

Docker creates virtual networking interfaces and bridges for container networking.

The current host has Docker bridge networking in addition to the host network used by Pi-hole.

---

## 6. Pi-hole Networking

Pi-hole runs using Docker host networking:

```yaml
network_mode: host
```

This is an important architectural choice.

Instead of:

```text
Client
  │
  ▼
Docker published port
  │
  ▼
Pi-hole container
```

the deployment effectively uses:

```text
Client
  │
  ▼
Host network namespace
  │
  ▼
Pi-hole container
```

The container therefore shares the host network namespace.

This allows Pi-hole to bind directly to host ports such as DNS port `53`.

It also means that Pi-hole's network exposure must be considered together with the host's network and OCI security controls.

---

## 7. DNS Listener

Pi-hole provides DNS service on:

```text
TCP/UDP 53
```

The service is configured to listen broadly at the application level.

This makes network-level access control particularly important.

An incorrectly exposed Pi-hole DNS listener could potentially become an unintended public DNS resolver.

Therefore:

> Pi-hole's `listeningMode` configuration must not be considered a substitute for network access control.

The OCI network configuration and any other applicable network security controls must prevent unauthorized Internet clients from reaching the DNS service.

---

## 8. Unbound Networking

Unbound is intentionally restricted to the local host interface:

```text
127.0.0.1:5335
```

Both UDP and TCP DNS are supported.

Conceptually:

```text
Pi-hole
   │
   │ localhost
   ▼
127.0.0.1:5335
   │
   ▼
Unbound
```

Because Unbound listens only on localhost, it is not intended to be directly reachable by remote clients.

This creates a useful network boundary between the public-facing DNS service and the recursive resolver.

---

## 9. Tailscale Access

Tailscale provides the private management path.

The intended administrative flow is:

```text
Administrator
      │
      │ encrypted Tailscale connection
      ▼
Tailscale overlay
      │
      ▼
Ubuntu server
      │
      ▼
SSH
```

The OCI public address is not required for normal SSH administration.

This reduces the need to expose SSH directly through the public Internet.

The repository does not contain:

* Tailscale IP addresses
* Tailscale node names
* User identities
* Authentication keys
* SSH private keys

---

## 10. Docker Networking

Docker provides multiple networking modes on the host.

The main Pi-hole container uses:

```text
host networking
```

Other containers can use normal Docker bridge networking.

For example:

```text
Ubuntu Host
│
├── ens3
│
├── tailscale0
│
├── docker0
│
├── Docker bridge network
│
├── Pi-hole
│    └── host network
│
├── Unbound
│    └── localhost
│
└── Monitoring services
     └── Docker networking / host networking
```

The exact dynamically generated Docker bridge identifiers are intentionally not documented because they are implementation details rather than stable architecture identifiers.

---

## 11. Network Separation

The deployment uses network separation to reduce unnecessary exposure.

### Public / OCI network

Used for:

* Server connectivity
* Internet-bound traffic
* OCI infrastructure communication

### Tailscale network

Used for:

* Private administrative access
* Trusted device connectivity

### Localhost

Used for:

* Pi-hole → Unbound communication
* Unbound's recursive DNS service

### Docker networks

Used for:

* Container-to-container communication where required
* Supporting application services

---

## 12. Effective Exposure

A service listening on an interface does not necessarily mean that it is Internet-accessible.

Effective exposure should be evaluated as:

```text
Service listener
      │
      ▼
Host networking / Docker rules
      │
      ▼
Host firewall
      │
      ▼
OCI Security List
      │
      ▼
OCI Network Security Group, if applicable
      │
      ▼
Routing
      │
      ▼
Internet
```

All applicable layers need to be considered before declaring a service publicly reachable or unreachable.

This is particularly important for DNS because Pi-hole listens on port `53`.

---

## 13. Current Security Boundary

The current architecture intentionally places the strongest trust boundaries around:

1. OCI network ingress
2. Tailscale private access
3. Localhost-only Unbound
4. Docker/container boundaries
5. Pi-hole application controls

UFW is **not** part of the current host firewall configuration.

The system should therefore not be described as relying on UFW for network protection.

---

## 14. Network Design Principles

The deployment follows several principles:

* Do not expose services unnecessarily.
* Use private overlay networking for administration where practical.
* Keep the recursive resolver local to the host.
* Separate DNS filtering from recursive DNS resolution.
* Treat application listeners and network access controls as separate security layers.
* Verify effective exposure instead of assuming it from a single configuration source.
* Avoid publishing infrastructure-specific addressing in a public repository.

---

## 15. Public Documentation Policy

The following information is intentionally generalized in this repository:

```text
Public IP addresses
Private IP addresses
Tailscale addresses
Device names
OCI resource identifiers
SSH hostnames
Authentication credentials
API tokens
Private keys
Certificates containing sensitive information
```

This allows the architecture to be publicly documented without unnecessarily exposing details of the live deployment.
