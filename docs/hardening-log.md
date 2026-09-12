# Security Hardening Log

This document records the security-hardening work performed against the running OCI-hosted Pi-hole deployment.

The scope is limited to the current deployment and the security changes already performed or currently under investigation.

This document is an operational hardening log. It is not the final security architecture or security specification.

The final security posture will be consolidated into `docs/security.md` after the hardening work is complete.

---

## Hardening Status

| Area | Status |
|---|---|
| External exposure | Verified |
| Unbound | Hardened |
| SSH | Hardened |
| Host services | Reduced |
| Beszel | Hardened |
| Pi-hole web ACL | Hardened |
| Pi-hole API | Hardened |
| NET_ADMIN | Under review |
| NTP | Under review |
| Docker capabilities | Under review |
| Listener bindings | Under review |

---

## 1. External Exposure

### What was checked

External connectivity was tested against the public interface from an external system.

The following TCP ports were tested:

- TCP/22 — SSH
- TCP/53 — DNS
- TCP/80 — HTTP
- TCP/443 — HTTPS
- TCP/8080 — Pi-hole web interface
- TCP/8090 — Beszel
- TCP/8443 — Pi-hole HTTPS

UDP/53 was also tested separately.

DNS queries were tested against the public DNS endpoint using both normal DNS and TCP DNS.

### Observed results

All tested TCP ports returned:

`filtered`

UDP/53 returned:

`open|filtered`

However, this did not result in a successful DNS response.

A direct DNS query against the public address timed out.

A TCP DNS query against the public address also timed out.

The external tests therefore did not demonstrate a publicly reachable DNS resolver.

Public SSH and web administration were also not reachable through the tested external path.

### Security conclusion

The external tests provide evidence that the tested services are not directly reachable through the tested public TCP paths.

UDP/53 must not be considered definitively closed based solely on the `open|filtered` result because UDP scanning cannot reliably distinguish a filtered port from an open port that does not respond to the probe.

OCI networking, host listeners, Docker networking and application-level controls remain separate security layers.

---

## 2. Unbound

### Finding

Unbound is used as the local recursive DNS resolver for Pi-hole.

The resolver is bound to the local host interface rather than being exposed as a public recursive resolver.

The host also contained `unbound-resolvconf.service`, which was repeatedly failing because the expected `systemd-resolved` integration was not present.

### Change performed

Disabled:

`unbound-resolvconf.service`

The service was stopped and disabled so that it no longer participates in the system's service lifecycle.

### Verification

After the change:

- Unbound remained enabled and active.
- The local Unbound listener remained available.
- Pi-hole continued resolving DNS through Unbound.
- DNS blocking continued to work.
- No failed systemd services remained.
- Direct DNSSEC testing against Unbound returned a response with the `AD` flag set.

### Security impact

The failing and unnecessary resolver-integration service was removed without affecting the DNS stack.

Unbound continues to provide the recursive resolver function, including DNSSEC validation.

---

## 3. SSH

### Finding

SSH was already restricted to key-based authentication with root login and password authentication disabled.

Additional SSH functionality was identified that was not required for the intended administration model.

### Changes performed

Disabled:

- `X11Forwarding`
- `AllowTcpForwarding`
- `AllowAgentForwarding`

All three are configured as:

`no`

### Verification

The effective configuration was verified using:

`sshd -T`

The effective configuration confirmed:

- root login disabled
- public-key authentication enabled
- password authentication disabled
- keyboard-interactive authentication disabled
- X11 forwarding disabled
- TCP forwarding disabled
- agent forwarding disabled

A fresh SSH connection through Tailscale was successfully tested after the change.

### Security impact

The SSH service retains the functionality required for administration while removing unnecessary forwarding features.

Administrative access continues through the intended Tailscale path.

---

## 4. Host Service Reduction

### Finding

The host is a headless OCI KVM/QEMU server.

Several enabled services were identified whose functionality was not required by the current deployment.

Each service was reviewed before being disabled.

### Changes performed

| Service | Action | Reason |
|---|---|---|
| `multipathd.service` | Disabled | No multipath devices or maps were present; the system uses a single OCI block volume |
| `ModemManager.service` | Disabled | No modem hardware or modem-related devices were present |
| `udisks2.service` | Disabled | Headless server with no desktop or removable-storage requirement |
| `open-vm-tools.service` | Disabled | Host uses KVM/QEMU rather than VMware |
| `apport.service` | Disabled | Crash-reporting service not required for this server deployment |

### Verification

After the service changes:

- Docker remained operational.
- Pi-hole remained healthy.
- DNS resolution continued to work.
- DNS blocking continued to work.
- Unbound remained operational.
- Tailscale remained operational.
- SSH remained operational.
- No failed systemd services remained.

### Intentionally retained

The following services were deliberately not removed because they may have an OCI, storage, boot, networking, security, container or system-management role:

- `iscsid`
- `nvmefc-boot-connections`
- `nvmf-autoconnect`
- `lvm2-monitor`
- `fwupd`
- `snapd`
- Oracle Cloud Agent
- cloud-init
- `vgauth`
- Docker / containerd
- AppArmor
- unattended-upgrades
- networking services
- time synchronization services

---

## 5. Docker Hardening

### Beszel Docker Socket

#### Finding

The Beszel agent originally had access to the Docker socket through a read-only bind mount:

`/var/run/docker.sock:/var/run/docker.sock:ro`

Although the mount was read-only, Docker socket access was considered security-sensitive because it exposes the Docker control interface to the container.

#### Change performed

The Docker socket mount was removed from the Beszel agent configuration.

The agent retains its required persistent data mount.

A pre-change configuration backup was created:

`compose.yaml.pre-docker-socket-hardening`

#### Verification

After the change:

- Beszel agent remained running.
- The agent retained its persistent data.
- The agent detected system disks and interfaces.
- The agent successfully established its WebSocket connection to the Beszel Hub.
- Docker remained operational.
- Pi-hole remained healthy.
- DNS resolution continued to work.
- DNS blocking continued to work.
- Unbound DNSSEC validation continued to work.

#### Security impact

Beszel no longer requires access to the host Docker socket.

This removes an unnecessary privileged control interface from the monitoring container.

---

## 6. Pi-hole Web Administration

### Finding

Pi-hole web administration was broadly reachable at the application listener level.

Because administration is intended to occur through Tailscale, an application-level access restriction provides an additional security boundary.

### Change performed

Configured Pi-hole FTL:

`webserver.acl`

The ACL allows:

- localhost
- Tailscale IPv4 clients
- Tailscale IPv6 clients

The exact live addresses and network details are intentionally not recorded in this repository.

### Verification

The following behavior was confirmed:

- localhost → allowed
- Tailscale client → allowed
- OCI private-interface access → rejected

The Pi-hole web interface remained accessible through Tailscale after the change.

### Security impact

Pi-hole web administration is now restricted at the application layer to the intended administrative networks.

This supplements the existing OCI and Tailscale controls.

---

## 7. Pi-hole API

### Finding

Pi-hole's destructive API operations were enabled.

These operations are not required for normal DNS resolution, blocking or basic administration.

### Change performed

Changed:

`webserver.api.allow_destructive`

from:

`true`

to:

`false`

A pre-change Pi-hole configuration backup was created:

`pihole.toml.pre-api-hardening`

### Verification

Verified:

`webserver.api.allow_destructive = false`

Post-change checks confirmed:

- Pi-hole container remained healthy.
- DNS resolution continued to work.
- DNS blocking continued to work.
- Unbound DNSSEC validation continued to work.
- Tailscale web administration continued to work.
- No failed systemd services remained.

### Security impact

The destructive API surface has been reduced.

Normal DNS operation and the required administrative workflow remain functional.

---

## 8. Remaining Capability Review

### NET_ADMIN

**Status: Under review**

The Pi-hole container currently retains:

`NET_ADMIN`

No change has been made yet.

The capability requires further investigation because Pi-hole uses host networking and also provides NTP functionality.

The objective is to establish whether the capability is actually required before removing it.

Removing a capability without establishing its requirement could break required Pi-hole functionality, so this remains a deliberate investigation rather than an assumed hardening change.

### NTP

**Status: Under review**

Pi-hole FTL currently provides NTP functionality.

The host itself has working system time synchronization.

Pi-hole logs indicate that its NTP client cannot set the system clock because `CAP_SYS_TIME` is not available, while the NTP server remains active.

No change has been made yet.

The requirement for Pi-hole NTP will be evaluated together with the `NET_ADMIN` review.

### Docker capabilities

**Status: Under review**

The remaining capabilities granted to the Pi-hole container have not yet been fully reviewed.

The next step is to determine which capabilities are actually required by the current Pi-hole configuration and whether any can be safely removed.

No capability will be removed solely for the purpose of reducing a count; each change will be validated against the required functionality.

---

## 9. Listener Binding Review

**Status: Under review**

The deployment currently uses Pi-hole host networking.

As a result, Pi-hole listeners are directly visible on the host network namespace.

The current Pi-hole DNS listener is broadly bound, while the web interface has an application-level ACL restricting access.

Listener bindings will be reviewed separately to determine whether individual services can be bound more narrowly without breaking the intended DNS and administration workflows.

No listener binding has been changed as part of this review yet.

---

## 10. Remaining Hardening Work

The following hardening work remains:

- [ ] Determine whether Pi-hole requires `NET_ADMIN`
- [ ] Determine whether Pi-hole NTP is required
- [ ] Review remaining Docker capabilities
- [ ] Review Pi-hole DNS and web listener bindings
- [ ] Perform final external exposure validation
- [ ] Review retained host services
- [ ] Verify the final runtime security posture
- [ ] Consolidate completed findings into `docs/security.md`
- [ ] Perform final repository information-disclosure review
- [ ] Decide whether the repository is ready to become public

No live configuration should be changed without the one-change → verify methodology used throughout this hardening process.

---

## 11. Final Consolidation

This document is an operational record of the hardening work.

Once the remaining hardening items have been reviewed and verified, the completed security posture will be consolidated into:

`docs/security.md`

The final public documentation must describe the security architecture and controls without exposing deployment-specific information.

The following information must not be committed to the public repository:

- public IP addresses
- private host addresses
- Tailscale device names
- credentials
- API keys
- private keys
- certificates containing sensitive material
- Pi-hole query logs
- DHCP lease information
- live runtime databases
- other deployment-specific secrets
