# Proxmox Monitoring LXC Hardening

## Purpose

This document records the security hardening applied to a Proxmox-hosted monitoring container running Grafana, Prometheus, and a Proxmox VE exporter.

The goal was to reduce attack surface, constrain network access, enforce least privilege, protect administrative sessions, and keep internal monitoring components isolated without changing the monitoring architecture unnecessarily.

> This public version is sanitized for a public Git repository. Internal IP addresses, account names, token identifiers, certificate locations, residual risks, and deferred security items are intentionally documented only in the private companion file.

---

## Architecture

```text
Approved admin systems
        |
        | HTTPS to Grafana only
        v
+-------------------------------+
| Monitoring LXC                |
|                               |
| Grafana         :3000 / HTTPS |
| Prometheus      127.0.0.1     |
| PVE Exporter    127.0.0.1     |
| Alertmanager    disabled      |
| Local SMTP      disabled      |
+-------------------------------+
        |
        | narrowly allowed monitoring traffic
        v
+-------------------------------+
| Proxmox VE host               |
| PVE API                       |
| Host Node Exporter            |
+-------------------------------+
```

The monitoring stack uses native systemd services rather than Docker or Podman.

---

## Security Objectives

- Expose only the Grafana web interface to approved administrative systems.
- Bind internal monitoring services to loopback.
- Prevent unrestricted inbound and outbound network access.
- Remove unused remote-management and hardware-management packages.
- Use least-privilege credentials for Proxmox API access.
- Verify TLS for API traffic.
- Harden services with systemd sandboxing.
- Protect Grafana authentication and session cookies with HTTPS.
- Automatically install Debian security updates.
- Keep unused notification services disabled until required.

---

## Container Isolation

Unnecessary LXC feature flags were disabled:

```bash
pct set <CT_ID> -features nesting=0,keyctl=0
pct reboot <CT_ID>
```

The container runs with host-enforced AppArmor and seccomp protections. Docker and Podman are not installed.

The guest nftables service was disabled so firewall policy is enforced centrally through the Proxmox firewall:

```bash
pct exec <CT_ID> -- systemctl disable --now nftables
```

---

## Attack-Surface Reduction

Unused services and packages were removed, including:

- OpenSSH server
- local Node Exporter
- unnecessary Node Exporter collector packages
- SMART/NVMe/IPMI utilities and orphaned dependencies

The container has no SSH listener. Administration is performed from the Proxmox host with `pct exec`.

Alertmanager and the local SMTP service remain installed and securely configured for future use, but are disabled while alert delivery is not in use.

---

## Network Hardening

### Proxmox firewall

The container firewall is enabled with default-deny policies:

```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: DROP
```

Inbound access is limited to:

- DHCP replies required by the container
- Grafana HTTPS from an approved Proxmox IP set

Outbound access is limited to the traffic required for:

- DHCP
- approved DNS resolvers
- the Proxmox API
- the Proxmox host's Node Exporter
- package and security updates

A sanitized example:

```ini
[RULES]
# DHCP reply
IN ACCEPT -p udp -sport 67 -dport 68

# Grafana from approved admin systems only
IN ACCEPT -source +dc/<ADMIN_IPSET> -p tcp -dport 3000

# DHCP request
OUT ACCEPT -p udp -sport 68 -dport 67

# Approved DNS resolvers
OUT ACCEPT -dest <PRIMARY_DNS> -p udp -dport 53
OUT ACCEPT -dest <PRIMARY_DNS> -p tcp -dport 53
OUT ACCEPT -dest <SECONDARY_DNS> -p udp -dport 53
OUT ACCEPT -dest <SECONDARY_DNS> -p tcp -dport 53

# Proxmox API
OUT ACCEPT -dest <PVE_HOST> -p tcp -dport 8006

# Host Node Exporter
OUT ACCEPT -dest <PVE_HOST> -p tcp -dport 9100

# Required repository traffic
OUT ACCEPT -p tcp -dport 80
OUT ACCEPT -p tcp -dport 443
```

No outbound SMTP rule is present while email alerting is disabled.

### Host firewall

The Proxmox host firewall permits the monitoring container to reach only the host services it must monitor:

- Proxmox API
- host Node Exporter

These rules are placed before broader LAN-drop rules.

---

## Service Binding

Internal components are not exposed to the LAN:

| Service | Listener |
|---|---|
| Grafana | HTTPS on TCP 3000, filtered by the Proxmox firewall |
| Prometheus | `127.0.0.1:9090` |
| PVE Exporter | `127.0.0.1:9221` |
| Alertmanager | Disabled; configured for loopback only |
| Local SMTP | Disabled |

Prometheus scrapes:

- the Proxmox host Node Exporter
- PVE Exporter runtime metrics
- Proxmox VE API metrics through the local PVE Exporter

---

## Proxmox API Least Privilege

The PVE Exporter uses a dedicated Proxmox API token with:

- privilege separation enabled
- an audit-only role
- no write permissions
- verified TLS

The exporter configuration is readable only by root and the exporter service group.

The Proxmox cluster CA was regenerated after validating that the previous CA did not include the X.509 extensions required by strict modern TLS clients. The monitoring container trusts the corrected public CA and verifies the Proxmox API certificate.

No API token secret is included in this repository.

---

## systemd Service Sandboxing

The PVE Exporter and Alertmanager were hardened with systemd restrictions such as:

```ini
[Service]
NoNewPrivileges=yes
CapabilityBoundingSet=
AmbientCapabilities=
RestrictSUIDSGID=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectHome=yes
ProtectSystem=strict
```

Additional restrictions were applied where compatible, including:

- restricted address families
- namespace restrictions
- kernel and control-group protection
- native system-call architecture enforcement
- read/write path allowlisting
- invisible `/proc` access where supported

Services were restarted and verified after applying the drop-ins.

---

## Grafana Hardening

### Access controls

- Anonymous access disabled
- User self-registration disabled
- Public dashboards disabled
- External snapshots disabled
- GUI plugin administration disabled
- No unsigned-plugin exceptions
- Only the approved administrative IP set can reach Grafana

### Administrator identity

A named administrative account was created and tested with:

- organization Admin role
- server-wide Grafana Admin permission

The original default administrator account was then:

- demoted
- logged out from all sessions
- disabled

### HTTPS

Grafana uses native HTTPS on its existing port with:

```ini
[server]
protocol = https
min_tls_version = TLS1.2
http_port = 3000
domain = <GRAFANA_INTERNAL_FQDN>
enforce_domain = true
root_url = https://<GRAFANA_INTERNAL_FQDN>:3000/
cert_file = <SERVER_CERTIFICATE>
cert_key = <SERVER_PRIVATE_KEY>
```

The certificate contains both the internal DNS name and the service IP as Subject Alternative Names.

The internal root CA is trusted by approved administrative systems. The CA private key is never installed in the container.

### Cookies and browser protections

```ini
[security]
cookie_secure = true
cookie_samesite = strict
strict_transport_security = true
strict_transport_security_max_age_seconds = 86400
strict_transport_security_preload = false
strict_transport_security_subdomains = false
```

Verified response headers include:

```text
Strict-Transport-Security: max-age=86400
X-Content-Type-Options: nosniff
X-Frame-Options: deny
```

---

## File Permissions

Sensitive files use restrictive ownership and permissions, including:

- Grafana configuration
- Grafana database
- Grafana TLS private key
- PVE Exporter credential configuration
- Alertmanager configuration

The CA private key remains outside the monitoring container.

---

## Updates

Debian security updates are installed automatically through `unattended-upgrades`.

Only Debian security origins are allowed for unattended installation. Third-party packages such as Grafana remain manually reviewed and updated.

Automatic rebooting is disabled.

---

## Monitoring Interval

Prometheus uses a one-minute scrape interval for this homelab environment.

Grafana's Prometheus data source must also have its scrape interval set to:

```text
1m
```

This ensures `$__rate_interval` is calculated correctly. Without this setting, rate-based panels can show `No data` even while gauge metrics continue to work.

---

## Verification Performed

The hardened state was validated with:

- Proxmox firewall compilation
- full TCP port scan from an approved system
- targeted checks of blocked management and monitoring ports
- outbound-deny testing
- DNS resolution tests
- package repository access tests
- PVE API exporter tests
- Prometheus target health checks
- certificate-chain validation
- HTTPS response and security-header checks
- system-wide CA trust validation without bypassing TLS
- Grafana administrator permission and login tests

Expected externally reachable service:

```text
TCP 3000 — Grafana HTTPS, approved systems only
```

Internal-only services remain unreachable from the LAN.

---

## Operational Notes

- Keep the internal DNS record for Grafana available to all approved administrative systems.
- Renew the Grafana server certificate before expiration.
- Preserve the private CA key in protected, encrypted storage.
- If Alertmanager and SMTP are enabled later, add only the exact outbound destination and port required for the relay.
- Review firewall policy whenever a new exporter, webhook, repository, or notification channel is introduced.
- Keep Grafana's configured scrape interval aligned with Prometheus.
- Do not add broad management access merely for convenience; use `pct exec` from the Proxmox host.

---

## Result

The monitoring container now has:

- minimized services and packages
- no SSH exposure
- default-deny inbound and outbound firewalling
- loopback-only internal monitoring components
- least-privilege, TLS-verified Proxmox API access
- systemd sandboxing
- secure Grafana administrator handling
- trusted HTTPS with secure cookies and HSTS
- automated Debian security updates

Environment-specific exceptions and residual risks are tracked separately in the private hardening notes.
