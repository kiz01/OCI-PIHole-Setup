# Rebuild Guide

This document describes how to manually rebuild the current OCI-hosted Pi-hole DNS filtering stack from a fresh Ubuntu host.

The procedure is based on the verified state of the existing deployment. It is intentionally manual: automation is not part of this phase.

> **Important:** Replace deployment-specific values such as Tailscale addresses, credentials, Beszel keys, hostnames, and OCI-specific identifiers with values from the new deployment. Never copy secrets from the existing server into this repository.

---

## 1. Scope

The rebuilt system consists of:

- Ubuntu Linux host on OCI
- Docker Engine and Docker Compose
- Tailscale for private administration/access
- Unbound as the local recursive DNS resolver
- Pi-hole as the DNS filtering layer
- Beszel Hub and Agent for monitoring
- SSH restricted to public-key authentication

The intended DNS path is:

```text
Authorized client
      |
      v
    Pi-hole
      |
      v
Unbound 127.0.0.1:5335
      |
      v
Authoritative DNS infrastructure
```

Administrative web access is intended to use the Tailscale network.

---

## 2. Architecture

### Host

The current deployment uses:

```text
OCI VM
  |
  +-- ens3
  |
  +-- Docker
  |
  +-- Tailscale
  |
  +-- Unbound
  |
  +-- Pi-hole
  |
  +-- Beszel
  |
  +-- SSH
```

### Docker projects

```text
/opt/docker/
├── piholeserver/
│   ├── compose.yaml
│   ├── etc-pihole/
│   └── etc-dnsmasq.d/
│
└── beszel/
    ├── compose.yaml
    ├── beszel_data/
    └── beszel_agent_data/
```

The existing `agent_data/` directory is not part of the active Beszel Compose configuration and is therefore not required for a rebuild.

---

## 3. Prerequisites

The current host was verified as:

- Ubuntu 24.04.4 LTS
- x86-64
- KVM/QEMU virtual machine
- Docker Engine 29.7.1
- Docker Compose 5.3.1
- Unbound 1.19.2
- Tailscale installed and active
- SSH server installed

The exact package installation procedure used when the current host was originally created was not captured in the inventory. Install the supported versions of the required software on the new host before continuing.

Required software:

```text
docker-ce
docker-ce-cli
containerd.io
docker-buildx-plugin
docker-compose-plugin
unbound
openssh-server
tailscale
```

---

## 4. OCI Instance & Network

The current deployment uses:

- OCI Mumbai region
- VCN: `Main-VCN`
- Public subnet: `Public-Subnet`
- Subnet CIDR: `10.0.1.0/24`
- VCN CIDR: `10.0.0.0/16`
- Internet Gateway for outbound connectivity

The current Security List has only ICMP ingress rules:

```text
0.0.0.0/0       ICMP type/code 3,4
10.0.0.0/16     ICMP type/code 3
```

There are no Security List ingress rules for:

```text
TCP 22
TCP/UDP 53
TCP 80
TCP 443
TCP 8080
TCP 8090
TCP 8443
```

The current deployment does not use an NSG.

### Network principle

Do not expose Pi-hole DNS, its web interface, SSH, or Beszel publicly unless there is an explicit architectural reason.

The current design relies heavily on OCI network controls in addition to host/application controls.

---

## 5. Ubuntu Base Setup

After provisioning the VM:

1. Update the operating system.
2. Confirm the expected hostname.
3. Confirm the timezone.
4. Confirm system time synchronization.
5. Install the required packages.
6. Confirm SSH access before making SSH hardening changes.

Expected timezone:

```text
Asia/Kolkata
```

Verify:

```bash
timedatectl
```

The host should report synchronized time and an active time synchronization service.

---

## 6. Docker

Create the project directories:

```bash
sudo mkdir -p /opt/docker/piholeserver
sudo mkdir -p /opt/docker/piholeserver/etc-pihole
sudo mkdir -p /opt/docker/piholeserver/etc-dnsmasq.d

sudo mkdir -p /opt/docker/beszel
sudo mkdir -p /opt/docker/beszel/beszel_data
sudo mkdir -p /opt/docker/beszel/beszel_agent_data
```

The existing deployment uses:

```text
/opt/docker/piholeserver
    ubuntu:ubuntu

/opt/docker/piholeserver/etc-pihole
    opc:opc

/opt/docker/piholeserver/etc-dnsmasq.d
    ubuntu:ubuntu

/opt/docker/beszel
    root:root

/opt/docker/beszel/beszel_data
    root:root

/opt/docker/beszel/beszel_agent_data
    root:root
```

The exact ownership on a new installation may depend on the chosen bootstrap process and should be verified before relying on it.

---

## 7. Tailscale

Install and authenticate Tailscale using the new deployment's authentication process.

Do not put an auth key or machine-specific Tailscale address in this repository.

The current architecture uses Tailscale for:

- private administration
- Pi-hole web access
- Beszel Agent → Hub communication

The current Pi-hole web ACL permits the Tailscale address space in addition to localhost.

After connecting the new host to Tailscale, record the new Tailscale address privately for deployment configuration.

---

## 8. Unbound

Install and enable Unbound.

The current deployment uses:

```text
Address: 127.0.0.1
Port:    5335
```

The effective configuration is:

```conf
server:
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-ip6: no
    do-udp: yes
    do-tcp: yes

    root-hints: "/var/lib/unbound/root.hints"

    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    qname-minimisation: yes
    prefetch: yes
    prefetch-key: yes

    edns-buffer-size: 1232

    rrset-cache-size: 64m
    msg-cache-size: 32m
    num-threads: 1

    access-control: 127.0.0.1/32 allow
    access-control: 0.0.0.0/0 refuse
```

The trust anchor configuration is:

```conf
server:
    auto-trust-anchor-file: "/var/lib/unbound/root.key"
```

Unbound remote control is enabled using the local control socket:

```text
/run/unbound.ctl
```

### Critical security property

Unbound must remain localhost-only.

Verify:

```bash
sudo ss -lntup | grep 5335
```

Expected:

```text
127.0.0.1:5335
```

Do not expose port 5335 through OCI, Docker, or Tailscale.

Validate the configuration:

```bash
sudo unbound-checkconf
```

---

## 9. Pi-hole

The current Pi-hole deployment uses:

```text
Image: pihole/pihole:2026.07.2
Network mode: host
Restart: unless-stopped
```

The active Compose configuration is conceptually:

```yaml
services:
  pihole:
    image: pihole/pihole:2026.07.2
    container_name: pihole
    network_mode: host
    restart: unless-stopped
    hostname: pihole

    environment:
      TZ: "Asia/Kolkata"
      FTLCONF_webserver_api_password: "<SET-OUT-OF-BAND>"
      FTLCONF_dns_upstreams: "127.0.0.1#5335"
      FTLCONF_webserver_port: "8080o,[::]:8080os,[::]:8443os"
      DNSMASQ_LISTENING: "local"

    volumes:
      - ./etc-pihole:/etc/pihole
      - ./etc-dnsmasq.d:/etc/dnsmasq.d
```

Do not place the real web/API password in this file.

### Important capability state

The current container does not add `NET_ADMIN`.

Do not reintroduce:

```yaml
cap_add:
  - NET_ADMIN
```

unless a future version of the deployment has a verified requirement for it.

The current container is also not privileged and has no Docker socket.

### DNS

Pi-hole listens on DNS port:

```text
53 TCP
53 UDP
```

The upstream resolver is:

```text
127.0.0.1#5335
```

The current Pi-hole DNS configuration includes:

```text
cache size:             10000
query logging:          enabled
bogus private:          enabled
stale cache:            3600
rate limit:             1000 queries / 60 seconds
```

### Adlist

The current deployment has one configured adlist:

```text
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
```

No custom hosts are currently configured.

### NTP

Pi-hole's built-in NTP server is disabled.

The current configuration is:

```text
ntp.ipv4.active = false
ntp.ipv6.active = false
```

This is intentional. The host's own time synchronization remains responsible for system time.

Verify that the Pi-hole container does not create a UDP/123 listener.

### Web interface

The current web configuration uses:

```text
8080
8443
```

with the Pi-hole web ACL restricted to:

- localhost
- Tailscale IPv4 address space
- Tailscale IPv6 address space

The exact Tailscale ranges/addresses are deployment-specific and must not be copied into a public configuration without review.

---

## 10. Generated Pi-hole State

Do not copy the existing Pi-hole data directory wholesale into a rebuild.

The following are runtime/generated state:

```text
gravity.db
gravity_old.db
gravity_backups/
listsCache/
pihole-FTL.db
pihole-FTL.db-shm
pihole-FTL.db-wal
dhcp.leases
config_backups/
migration_backup/
```

The generated `dnsmasq.conf` should also not be treated as the primary configuration source.

Pi-hole generates it from its managed configuration.

The current `etc-dnsmasq.d/` directory is empty.

### Configuration worth preserving

The intentional configuration inputs are:

```text
adlists.list
pihole.toml settings
custom DNS configuration, if any is intentionally added later
```

The current custom hosts file contains no entries.

---

## 11. Beszel

The current Hub configuration is:

```yaml
services:
  beszel:
    image: henrygd/beszel:latest
    container_name: beszel
    restart: unless-stopped

    ports:
      - "8090:8090"

    volumes:
      - ./beszel_data:/beszel_data
```

The current Agent configuration is:

```yaml
  beszel-agent:
    image: henrygd/beszel-agent
    container_name: beszel-agent
    restart: unless-stopped

    network_mode: host

    volumes:
      - ./beszel_agent_data:/var/lib/beszel-agent

    environment:
      LISTEN: 45876
      KEY: "<SET-OUT-OF-BAND>"
      HUB_URL: "http://<TAILSCALE-HUB-IP>:8090"
```

Replace:

```text
<TAILSCALE-HUB-IP>
<SET-OUT-OF-BAND>
```

with deployment-specific values.

### Security state

Both containers are:

```text
Privileged: false
CapAdd: none
Docker socket: absent
```

Do not add:

```text
/var/run/docker.sock
```

to the Agent.

The current Agent communicates with the Hub over the Tailscale network.

---

## 12. SSH Hardening

The effective SSH policy is:

```text
Port 22
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitTTY yes
PrintMotd no
```

The current server has authorized-key files for the existing administrative accounts.

On a rebuild, install only the keys required for the new deployment.

Never commit private keys or live `authorized_keys` contents to this repository.

Verify the effective configuration with:

```bash
sudo sshd -T | grep -Ei \
'^(port|listenaddress|permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|usepam|x11forwarding|allowtcpforwarding|allowagentforwarding|permittty|printmotd)'
```

---

## 13. Host Service Hardening

The current deployment intentionally disabled/stopped these services:

```text
multipathd
ModemManager
udisks2
open-vm-tools
apport
unbound-resolvconf
```

These were reviewed as unnecessary for the current server workload.

The following were intentionally retained because they remain part of the host/runtime environment or their removal was not justified by the current deployment:

```text
containerd
docker
apparmor
unattended-upgrades
cloud-init
Oracle Cloud Agent
snapd
vgauth
network/time services
iscsid
lvm2-monitor
fwupd
```

Do not blindly disable additional services simply to reduce the service count. Each removal should have a demonstrated requirement and be verified after the change.

---

## 14. Verification

Run the following after the rebuild.

### System

```bash
systemctl --failed
timedatectl
```

Expected:

```text
0 failed units
```

and synchronized system time.

### Docker

```bash
sudo docker ps
sudo docker compose ls
```

Expected active services:

```text
pihole
beszel
beszel-agent
```

### Pi-hole

```bash
sudo docker inspect pihole \
  --format 'Privileged={{.HostConfig.Privileged}} CapAdd={{json .HostConfig.CapAdd}}'

sudo docker logs --tail 100 pihole
```

Expected:

```text
Privileged=false
CapAdd=null
```

### Listeners

```bash
sudo ss -lntup
```

Verify:

```text
127.0.0.1:5335      Unbound
*:53                Pi-hole DNS
*:22                SSH
*:8080              Pi-hole web
*:8090              Beszel Hub
```

and confirm there is no UDP/123 listener.

### DNS

From an authorized client:

```bash
dig example.com @<PI-HOLE-IP>
```

Then verify blocking with a known blocked test domain.

### Unbound / DNSSEC

```bash
dig example.com @<PI-HOLE-IP> +dnssec
```

A successful DNSSEC-enabled response should include the `ad` flag when appropriate.

### Tailscale

Verify:

```bash
tailscale status
```

Then test:

```text
SSH → server
Pi-hole web → server:8080
Beszel → server:8090
```

using the Tailscale network.

### Pi-hole web ACL

Confirm that authorized Tailscale access works while an unauthorized network path cannot reach the administration interface.

### External exposure

Verify OCI ingress rules and externally test the intended public surface.

Do not rely only on host-side `ss` output. OCI security controls are part of this architecture.

---

## 15. Secrets & Deployment-Specific Values

Never commit:

```text
Pi-hole passwords
Beszel Agent keys
Tailscale authentication keys
SSH private keys
TLS private keys
API tokens
OCI credentials
OCI OCIDs
Public/private deployment IPs where disclosure is unnecessary
Device-specific identifiers
```

Use placeholders in public documentation:

```text
<PIHOLE_PASSWORD>
<BESZEL_AGENT_KEY>
<TAILSCALE_IP>
<OCI_INSTANCE_ID>
<SSH_PUBLIC_KEY>
```

Secrets should be supplied out-of-band during deployment.

---

## 16. Files That Must Not Be Copied

Do not copy the existing server's runtime data wholesale.

Especially avoid:

```text
/etc/pihole/tls.pem
/etc/pihole/tls.crt
/etc/pihole/tls_ca.crt

/etc/pihole/cli_pw

/etc/pihole/gravity.db
/etc/pihole/pihole-FTL.db*
/etc/pihole/dhcp.leases
/etc/pihole/listsCache/
/etc/pihole/config_backups/
/etc/pihole/gravity_backups/

/opt/docker/beszel/beszel_data/
/opt/docker/beszel/beszel_agent_data/
```

The last two directories contain application state and should only be restored intentionally when a data migration is actually required.

A clean rebuild should regenerate application state.

---

## 17. Final Security Checklist

Before considering the rebuild complete:

- [ ] OCI ingress rules reviewed
- [ ] No unnecessary public TCP services exposed
- [ ] SSH password authentication disabled
- [ ] Root SSH login disabled
- [ ] SSH forwarding disabled
- [ ] Tailscale access verified
- [ ] Unbound listens only on `127.0.0.1:5335`
- [ ] Unbound DNSSEC trust anchor configured
- [ ] Pi-hole DNS upstream points to Unbound
- [ ] Pi-hole web ACL permits only intended management networks
- [ ] Pi-hole NTP server disabled
- [ ] Pi-hole does not require `NET_ADMIN`
- [ ] Beszel Agent has no Docker socket
- [ ] No container is privileged
- [ ] Runtime secrets are outside Git
- [ ] Generated databases/caches are outside Git
- [ ] No unexpected host listeners
- [ ] No UDP/123 listener
- [ ] DNS resolution works
- [ ] DNS blocking works
- [ ] DNSSEC validation works
- [ ] `systemctl --failed` reports no failed units
- [ ] Repository secret scan completed

---

## 18. Rebuild Philosophy

This repository should describe the **desired reproducible architecture**, not preserve a snapshot of the current server.

The goal is:

```text
Fresh OCI VM
    ↓
Base host
    ↓
Docker + Tailscale + Unbound
    ↓
Pi-hole + Beszel
    ↓
Hardening
    ↓
Verification
```

Generated runtime state should be recreated by the services.

Deployment-specific secrets and identifiers should be supplied separately.

Automation should only be introduced after this manual rebuild procedure has been tested successfully on a fresh environment.
