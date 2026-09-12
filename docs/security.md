# Security Model

## 1. Purpose

This document describes the security model of the OCI Pi-hole deployment.

The objective is not to claim that the system is completely secure, but to document:

* Trust boundaries
* Network exposure
* Access controls
* Container security
* DNS security
* Secret handling
* Known risks
* Current limitations
* Security assumptions

The deployment uses multiple security layers rather than relying on a single control.

---

## 2. Security Architecture

The main security boundaries are:

```text id="x7lq1f"
                    Internet
                       │
                       ▼
              ┌─────────────────┐
              │   OCI Network   │
              │ Access Controls │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Ubuntu Server   │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
     Tailscale      Docker       Unbound
     private path   services     localhost
                       │
                       ▼
                    Pi-hole
```

Each layer has a different responsibility.

---

## 3. Threat Model

The deployment considers several classes of threats.

### Unauthorized network access

An attacker may attempt to connect to services exposed by the OCI instance.

Primary mitigations include:

* OCI network access controls
* Private Tailscale administration
* Avoiding unnecessary public service exposure
* Localhost-only Unbound

### DNS abuse

An incorrectly exposed DNS service could be abused as a public resolver.

This is particularly important because Pi-hole is configured to listen broadly.

The network layer must therefore prevent unauthorized clients from reaching DNS port `53`.

### Credential compromise

Credentials used by Pi-hole, monitoring services, SSH, or other components could provide access to the system if exposed.

Secrets must therefore remain outside version control.

### Container compromise

A vulnerable or compromised container could potentially be used as a stepping stone toward the host.

Container permissions and host-mounted resources must therefore be treated as security-sensitive.

### Configuration exposure

Publishing the live configuration without sanitization could reveal:

* Private network addresses
* Authentication credentials
* Tokens
* Private keys
* Internal hostnames
* Operational information

Public documentation must therefore use sanitized examples.

---

## 4. Network Security

The server is hosted in an OCI public subnet.

A public subnet provides a route to the Internet but does not automatically make every service publicly accessible.

Effective exposure depends on the combination of:

```text id="zq7z2x"
OCI routing
     │
     ▼
OCI Security List / NSG
     │
     ▼
Host networking
     │
     ▼
Docker networking
     │
     ▼
Service listeners
```

All relevant layers must be evaluated before considering a service exposed.

---

## 5. SSH Security

SSH is available on the host but administrative access is intended to occur through the Tailscale private network.

The normal administration path is:

```text id="m25u4h"
Administrator
      │
      ▼
Tailscale
      │
      ▼
Server
      │
      ▼
SSH
```

The public repository contains no:

* SSH private keys
* SSH passwords
* Tailscale authentication credentials
* Device-specific Tailscale information

Using a private overlay network reduces the need to expose SSH directly to the public Internet.

---

## 6. DNS Exposure

DNS is the most important network-facing security consideration.

Pi-hole listens on TCP and UDP port `53`.

The application-level configuration permits broad interface binding.

This means the security boundary cannot rely solely on Pi-hole's listener configuration.

The intended architecture is:

```text id="q6g7pu"
Trusted DNS client
       │
       ▼
Network access controls
       │
       ▼
Pi-hole :53
       │
       ▼
Unbound localhost:5335
```

### Open resolver risk

If unauthorized Internet clients were able to reach port `53`, the server could potentially be abused as a DNS resolver.

Potential consequences include:

* DNS query abuse
* Resource consumption
* Reputation damage
* Participation in DNS-based abuse
* Increased attack surface

Therefore, preventing unauthorized access to port `53` is a critical security requirement.

---

## 7. Unbound Security

Unbound is restricted to the local host interface:

```text id="2p9k7u"
127.0.0.1:5335
```

The configuration also explicitly refuses non-local IPv4 clients.

This provides an important isolation boundary:

```text id="z6xw6u"
Remote network
     │
     X
     │
     └── Unbound

Pi-hole
     │
     ▼
127.0.0.1:5335
     │
     ▼
Unbound
```

Remote clients should not communicate directly with the recursive resolver.

---

## 8. DNSSEC

DNSSEC validation is performed by Unbound.

The configuration uses an automatic trust-anchor file for DNSSEC validation.

This provides cryptographic validation of DNS data where DNSSEC is available.

The roles remain separated:

```text id="cbh7p8"
Pi-hole
  │
  ├── Filtering
  ├── Blocklists
  └── Query handling
          │
          ▼
       Unbound
          │
          ├── Recursive resolution
          └── DNSSEC validation
```

DNSSEC protects DNS data integrity/authenticity where applicable; it does not replace network access controls or endpoint security.

---

## 9. DNS Hardening

Unbound has several hardening features enabled, including:

* `harden-glue`
* `harden-dnssec-stripped`
* QNAME minimisation
* DNSSEC key prefetching
* Response prefetching
* Controlled EDNS buffer size

These settings reduce certain classes of DNS manipulation, leakage, or operational problems.

They should be considered supporting controls rather than a complete DNS security solution.

---

## 10. DNS Bypass

Pi-hole has protections enabled for several mechanisms that can bypass conventional DNS filtering.

These include protections related to:

* Firefox automatic DNS-over-HTTPS
* Apple iCloud Private Relay
* Discovery of Designated Resolvers

However, DNS-based enforcement has inherent limitations.

A device or application that uses its own encrypted DNS mechanism may be able to bypass network DNS filtering unless additional endpoint or network controls are implemented.

Therefore:

> Pi-hole should be treated as a DNS filtering layer, not as a universal traffic enforcement mechanism.

---

## 11. Docker Security

Pi-hole runs using Docker host networking.

This is operationally useful because DNS services can bind directly to host ports, but it reduces some of the network isolation normally provided by Docker bridge networking.

The container therefore shares the host network namespace.

This means container compromise must be considered in the context of host networking and the permissions granted to the container.

---

## 12. Pi-hole Capabilities

The Pi-hole container currently has:

```yaml id="t2l2mz"
cap_add:
  - NET_ADMIN
```

`NET_ADMIN` is a privileged Linux capability relative to an otherwise restricted container.

It should therefore be considered part of the Pi-hole container's security boundary.

Capabilities should only be granted when required by the application.

The current configuration is documented as-is rather than being changed as part of this documentation project.

---

## 13. Monitoring Stack

Beszel is deployed as a supporting monitoring service.

The Beszel agent has access to:

```text id="2o0t9a"
/var/run/docker.sock
```

with read-only access from the container.

Although the mount is read-only, Docker's Unix socket is highly sensitive because access to the Docker API can provide significant control over the container runtime depending on the API permissions available.

Therefore:

> The Docker socket mount is treated as a high-value security consideration.

The monitoring service should be kept appropriately restricted and trusted.

---

## 14. Secrets Management

Secrets are deliberately excluded from the repository.

Examples include:

* Pi-hole web/API credentials
* SSH private keys
* Tailscale authentication material
* Monitoring keys
* Monitoring tokens
* Monitoring hub URLs containing sensitive information
* Private TLS keys
* Other service credentials

Configuration examples should use placeholders:

```text id="2x1f3a"
YOUR_SECRET_HERE
YOUR_TOKEN_HERE
YOUR_PRIVATE_ADDRESS
```

Never replace placeholders with live credentials before committing.

---

## 15. Sensitive Runtime Data

The Pi-hole data directory contains generated and runtime information.

Examples include:

* DNS databases
* Query databases
* WAL/SHM files
* DHCP lease files
* Generated blocklist databases
* Cache files
* Configuration backups
* TLS material
* Runtime state

These files should not be copied wholesale into a public repository.

The repository should contain only intentionally sanitized configuration and documentation.

---

## 16. Public Repository Security

Before every commit, check for sensitive material.

Useful checks include:

```bash id="b8b5q8"
git status
```

```bash id="p0x5qn"
git diff --cached
```

```bash id="4g2m9a"
git ls-files
```

Search for obvious credential patterns before publishing:

```bash id="j8w8e2"
grep -RniE 'password|passwd|token|secret|private.key|BEGIN .*PRIVATE KEY' . \
  --exclude-dir=.git
```

This is not a replacement for dedicated secret-scanning tools, but it provides a basic manual check.

---

## 17. Information Disclosure

The public repository intentionally avoids publishing live infrastructure details.

The following should remain private:

```text id="6tqkhy"
Public IP addresses
Private IP addresses
Tailscale addresses
Device names
OCI resource identifiers
SSH configuration containing sensitive hosts
Credentials
API tokens
Private keys
Private certificates
Runtime databases
Query logs
DHCP lease information
```

Generic examples and placeholders should be used instead.

---

## 18. Host Security

The host operating system remains part of the security boundary.

Important operational controls include:

* Keeping Ubuntu updated
* Keeping Docker updated
* Keeping Pi-hole updated
* Keeping Unbound updated
* Reviewing exposed services
* Monitoring system health
* Reviewing authentication activity
* Maintaining backups
* Removing unnecessary services
* Using least privilege where practical

Updates should be tested and applied according to the operator's maintenance process.

---

## 19. Firewall Considerations

UFW is currently inactive on the host.

Therefore, UFW is **not** considered part of the current security boundary.

The network security model instead depends on the applicable OCI networking controls, Tailscale access model, service binding, and Docker/host networking behavior.

This distinction is important when reproducing the architecture.

A future deployment may choose to add host-level firewall controls, but doing so would represent a change to the documented architecture.

---

## 20. Current Security Limitations

The current deployment has several limitations that should be understood.

### DNS enforcement is not universal

Applications using alternative encrypted DNS mechanisms may bypass conventional DNS filtering.

### Pi-hole uses host networking

Host networking reduces Docker network isolation.

### Pi-hole has NET_ADMIN

The container has an additional Linux capability that increases its privilege relative to a capability-minimized container.

### Beszel accesses the Docker socket

The monitoring agent has read-only access to the Docker socket, which remains a sensitive host interface.

### Broad Pi-hole listener

Pi-hole's broad listener configuration requires effective network-layer access controls to prevent unintended DNS exposure.

### No host UFW

The host does not currently rely on UFW for firewalling.

These are documented characteristics of the current system rather than claims that the system is fully hardened.

---

## 21. Security Verification

Security should be verified from the outside as well as from the configuration.

Useful checks include:

### Listening services

```bash
sudo ss -tulpn
```

### Docker containers

```bash
docker ps
```

### Tailscale status

```bash
sudo tailscale status
```

### Unbound status

```bash
sudo systemctl status unbound
```

### Host firewall status

```bash
sudo ufw status verbose
```

### OCI network configuration

Review:

* Route tables
* Security Lists
* Network Security Groups
* VNIC attachments
* Subnet configuration

The effective security posture should be determined from the combination of these controls.

---

## 22. Security Review Checklist

Before exposing the system to a new network or changing its architecture:

* [ ] Verify OCI ingress rules
* [ ] Verify Network Security Groups, if used
* [ ] Verify host listeners
* [ ] Verify Docker published ports
* [ ] Verify Pi-hole DNS exposure
* [ ] Verify Unbound remains localhost-only
* [ ] Verify SSH access path
* [ ] Verify Tailscale access
* [ ] Check container capabilities
* [ ] Review Docker socket mounts
* [ ] Check for secrets before committing
* [ ] Review `.gitignore`
* [ ] Test DNS from an authorized client
* [ ] Confirm unauthorized DNS access is blocked
* [ ] Review logs for unexpected activity

---

## 23. Security Philosophy

The deployment follows a simple principle:

> **Do not assume that a service is secure because it is running correctly. Verify who can reach it, what privileges it has, and what happens if it is compromised.**

The architecture therefore treats:

* Network access
* DNS filtering
* Recursive resolution
* Container isolation
* Authentication
* Monitoring
* Secret management

as separate security concerns.

This makes the system easier to audit and provides a clearer foundation for future hardening.
