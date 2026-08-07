# Proxmox Monitoring LXC: LXC-Compatible Service Hardening and Recovery

## Why this work was necessary

After a Proxmox host kernel upgrade and reboot, the monitoring container started successfully, but several monitoring services failed immediately with `status=226/NAMESPACE`.

The affected services were not failing because of bad application configuration, missing binaries, invalid credentials, or corrupt data. They were failing because their systemd units attempted to create private mount, user, device, or related namespaces inside an AppArmor-confined LXC container.

The monitoring container was intentionally configured with a restrictive Proxmox LXC security posture:

- AppArmor confinement remained enabled.
- LXC nesting remained disabled.
- Keyctl remained disabled.
- The container was not converted to an unconfined or privileged workload.
- Services continued to run as dedicated unprivileged users.

The safest fix was therefore **not** to weaken the entire container. Instead, the incompatible inner systemd namespace controls were disabled only for the affected services, while process-level hardening and the outer LXC isolation boundary were retained.

This document records the root cause, design decisions, configuration changes, verification steps, rollback guidance, and upgrade considerations.

---

## Scope

This procedure covers the monitoring stack running inside a Debian-based Proxmox LXC container:

- Prometheus
- Grafana
- Prometheus PVE Exporter
- Proxmox host node exporter integration
- Internal HTTPS trust for Grafana
- Cleanup of redundant or container-incompatible systemd units

It does not cover initial installation of Prometheus or Grafana, dashboard provisioning, firewall policy, or creation of the Proxmox API account used by the exporter.

---

## Symptoms

After reboot, the monitoring container was running, but monitoring services were unavailable.

Typical service status:

```text
prometheus.service             failed
grafana-server.service         failed
pve-exporter.service           failed
```

Typical journal entries:

```text
Failed to set up mount namespacing: Permission denied
Failed at step NAMESPACE spawning ...: Permission denied
Main process exited, status=226/NAMESPACE
```

The host kernel log also showed AppArmor denying namespace and mount operations for the LXC profile:

```text
apparmor="DENIED" operation="userns_create"
apparmor="DENIED" operation="mount"
```

Application-specific validation showed that:

- The Prometheus configuration passed `promtool check config`.
- The PVE exporter executable and Python virtual environment were intact.
- The exporter configuration file had correct ownership and permissions.
- Grafana data, log, and configuration directories had correct ownership.
- The applications started normally when the incompatible namespace sandbox was omitted.

---

## Root cause

Several systemd security directives require systemd to construct an inner namespace or remount parts of the filesystem before starting the service.

The conflicting directives included:

```ini
PrivateTmp=yes
PrivateDevices=yes
PrivateUsers=yes
ProtectHome=yes
ProtectSystem=full
ProtectSystem=strict
ProtectControlGroups=yes
ProtectKernelModules=yes
ProtectKernelTunables=yes
ProtectClock=yes
ProtectHostname=yes
ProtectKernelLogs=yes
ProtectProc=invisible
```

Inside this Proxmox LXC, AppArmor correctly denied those inner namespace and mount operations.

The problem was therefore a **layering conflict**:

1. Proxmox LXC supplied the outer container isolation boundary.
2. AppArmor enforced container-level restrictions.
3. systemd attempted to create additional inner namespace isolation.
4. The LXC profile denied those operations.
5. The service failed before its application binary could start.

---

## Security decision

The following options were rejected:

- Enabling LXC nesting solely to satisfy service-level sandboxing
- Disabling AppArmor
- Making the container unconfined
- Converting the container to a privileged workload
- Broadly granting mount or namespace permissions
- Replacing package-owned unit files directly

Instead, service-specific drop-ins were used to disable only the incompatible controls.

The following protections were retained where compatible:

- Dedicated unprivileged service accounts
- `NoNewPrivileges=yes`
- Empty capability bounding sets
- Empty ambient capability sets
- `RestrictSUIDSGID=yes` where supported
- `LockPersonality=yes`
- `RestrictRealtime=yes`
- `RemoveIPC=yes`
- Restricted address families
- Native syscall architecture restriction
- Restriction of namespace creation by the service
- Existing Proxmox LXC AppArmor confinement
- Existing Proxmox firewall policy

---

# Implementation

## 1. PVE exporter compatibility and hardening

Create the drop-in directory:

```bash
install -d -m 0755 /etc/systemd/system/pve-exporter.service.d
```

Create:

```text
/etc/systemd/system/pve-exporter.service.d/security.conf
```

with:

```ini
[Service]
User=pve-exporter
Group=pve-exporter

NoNewPrivileges=yes
CapabilityBoundingSet=
AmbientCapabilities=
RestrictSUIDSGID=yes
LockPersonality=yes
RestrictRealtime=yes
RemoveIPC=yes
UMask=0077

RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6

RuntimeDirectory=pve-exporter
RuntimeDirectoryMode=0750
WorkingDirectory=/run/pve-exporter
Environment="HOME=/run/pve-exporter"
Environment="REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt"

PrivateTmp=no
PrivateDevices=no
ProtectHome=no
ProtectSystem=no

ExecStart=
ExecStart=/opt/pve-exporter/venv/bin/pve_exporter --config.file=/etc/prometheus/pve.yml --web.listen-address=127.0.0.1:9221

Restart=on-failure
RestartSec=5s
```

### Why the runtime directory was added

The exporter account intentionally used a nonexistent home directory. Gunicorn attempted to create its control socket relative to that home and logged a permission error.

`RuntimeDirectory=pve-exporter` creates a service-owned directory under `/run`, and the `HOME` and `WorkingDirectory` settings give Gunicorn a safe ephemeral location without creating a persistent interactive home directory.

Apply and restart:

```bash
systemctl daemon-reload
systemctl reset-failed pve-exporter.service
systemctl restart pve-exporter.service
```

Verify:

```bash
systemctl is-enabled pve-exporter.service
systemctl is-active pve-exporter.service
ss -lntp | grep '127.0.0.1:9221'
ls -ld /run/pve-exporter
```

Expected:

```text
enabled
active
```

The exporter must listen only on loopback:

```text
127.0.0.1:9221
```

---

## 2. Prometheus LXC compatibility drop-in

Create:

```text
/etc/systemd/system/prometheus.service.d/lxc-compat.conf
```

with:

```ini
[Service]
PrivateDevices=no
PrivateTmp=no
PrivateUsers=no
ProtectControlGroups=no
ProtectHome=no
ProtectKernelModules=no
ProtectKernelTunables=no
ProtectSystem=no

NoNewPrivileges=yes
CapabilityBoundingSet=
AmbientCapabilities=
LockPersonality=yes
MemoryDenyWriteExecute=yes
RemoveIPC=yes
RestrictNamespaces=yes
RestrictRealtime=yes
SystemCallArchitectures=native
```

The package-provided service account, executable, environment file, and `ExecStart` were left unchanged.

Prometheus remained bound to loopback through its package defaults:

```text
--web.listen-address=127.0.0.1:9090
```

Apply and restart:

```bash
systemctl daemon-reload
systemctl reset-failed prometheus.service
systemctl restart prometheus.service
```

Validate configuration and readiness:

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -fsS http://127.0.0.1:9090/-/ready
```

Expected:

```text
Prometheus Server is Ready.
```

A brief HTTP `503` during startup is normal while Prometheus initializes. Readiness checks should retry for a short period rather than treating the first `503` as a permanent failure.

---

## 3. Grafana LXC compatibility drop-in

Create:

```text
/etc/systemd/system/grafana-server.service.d/lxc-compat.conf
```

with:

```ini
[Service]
PrivateDevices=no
PrivateTmp=no
ProtectClock=no
ProtectControlGroups=no
ProtectHome=no
ProtectHostname=no
ProtectKernelLogs=no
ProtectKernelModules=no
ProtectKernelTunables=no
ProtectProc=default
ProtectSystem=no

NoNewPrivileges=yes
CapabilityBoundingSet=
AmbientCapabilities=
LockPersonality=yes
RemoveIPC=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
RestrictNamespaces=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
SystemCallArchitectures=native
UMask=0027
```

Apply and restart:

```bash
systemctl daemon-reload
systemctl reset-failed grafana-server.service
systemctl restart grafana-server.service
```

### HTTPS health-check requirement

Grafana was configured for HTTPS. A plain HTTP request to port 3000 produced repeated HTTP `400` responses and journal messages such as:

```text
client sent an HTTP request to an HTTPS server
```

Health checks must therefore use HTTPS and the configured hostname:

```bash
curl -fsS \
  --resolve <GRAFANA_FQDN>:3000:127.0.0.1 \
  https://<GRAFANA_FQDN>:3000/api/health
```

Expected response:

```json
{
  "database": "ok",
  "version": "...",
  "commit": "..."
}
```

Do not use `-k` as the permanent health check. It is acceptable only as a temporary diagnostic tool while validating certificate trust.

---

## 4. Retire redundant or container-incompatible units

The following units failed because they attempted mounts or namespaces that were already supplied or controlled by the LXC environment:

```text
dev-hugepages.mount
dev-mqueue.mount
run-lock.mount
tmp.mount
systemd-logind.service
systemd-networkd.service
systemd-networkd.socket
```

### Mount units

The relevant directories already existed with usable permissions:

```text
/dev/hugepages  0755
/dev/mqueue     0755
/run/lock       1777
/tmp            1777
```

The failed units were not required to make those paths usable. They were repeatedly attempting to mount filesystems that the container environment already exposed or controlled.

### Network manager

The container used Debian `ifupdown`:

```text
networking.service: active
```

Its interface configuration used `/etc/network/interfaces`. `systemd-networkd` was disabled and redundant, but its socket was still enabled and attempted to activate the service.

### Login manager

The monitoring appliance had:

- No SSH service
- No active login sessions
- No requirement for desktop seats or interactive user session management

`systemd-logind` was therefore unnecessary for this appliance role.

Disable and mask:

```bash
systemctl disable --now \
  systemd-networkd.socket \
  systemd-networkd.service || true

systemctl mask \
  dev-hugepages.mount \
  dev-mqueue.mount \
  run-lock.mount \
  tmp.mount \
  systemd-logind.service \
  systemd-networkd.service \
  systemd-networkd.socket

systemctl reset-failed
```

Masking `tmp.mount` does not remove `/tmp`. It only prevents systemd from attempting the incompatible tmpfs mount. `/tmp` remains a root-owned sticky directory with mode `1777`.

Verify:

```bash
systemctl is-active networking.service
ip -brief address
ip route
getent hosts deb.debian.org
systemctl --failed --no-pager
```

Expected:

```text
0 loaded units listed.
```

---

## 5. Install the internal root CA

Grafana used a certificate issued by the homelab root CA. The Grafana server certificate was correct, but the monitoring container initially lacked the public root CA certificate in its trust store.

Copy only the public root certificate to:

```text
/usr/local/share/ca-certificates/<homelab-root-ca>.crt
```

Set ownership and permissions:

```bash
chown root:root /usr/local/share/ca-certificates/<homelab-root-ca>.crt
chmod 0644 /usr/local/share/ca-certificates/<homelab-root-ca>.crt
```

Update the trust store:

```bash
update-ca-certificates
```

The following warning is normal because `ca-certificates.crt` is a combined bundle rather than a single certificate:

```text
rehash: warning: skipping ca-certificates.crt, it does not contain exactly one certificate or CRL
```

Validate the installed root certificate:

```bash
openssl verify \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  /usr/local/share/ca-certificates/<homelab-root-ca>.crt
```

Expected:

```text
/usr/local/share/ca-certificates/<homelab-root-ca>.crt: OK
```

Then validate Grafana without an insecure bypass:

```bash
curl -fsS \
  --resolve <GRAFANA_FQDN>:3000:127.0.0.1 \
  https://<GRAFANA_FQDN>:3000/api/health
```

Never copy the CA private key into the monitoring container.

---

# Final verification

## Service state

```bash
for service in \
  prometheus.service \
  grafana-server.service \
  pve-exporter.service
do
  printf '%-30s ' "$service"
  systemctl is-active "$service"
done
```

Expected:

```text
prometheus.service             active
grafana-server.service         active
pve-exporter.service           active
```

## Listening sockets

```bash
ss -lntp | grep -E ':(3000|9090|9221)\b'
```

Expected design:

| Service | Expected listener |
|---|---|
| Prometheus | `127.0.0.1:9090` |
| PVE exporter | `127.0.0.1:9221` |
| Grafana | HTTPS on port `3000`, restricted externally by Proxmox firewall |

## Prometheus targets

```bash
curl -fsSG \
  --data-urlencode 'query=up' \
  http://127.0.0.1:9090/api/v1/query
```

Each configured monitoring target should return value `1`.

## Failed units

```bash
systemctl --failed --no-pager
```

Expected:

```text
0 loaded units listed.
```

## Reboot persistence

After the next planned maintenance reboot, repeat:

```bash
systemctl is-active prometheus.service
systemctl is-active grafana-server.service
systemctl is-active pve-exporter.service
systemctl --failed --no-pager
curl -fsS http://127.0.0.1:9090/-/ready
```

Also verify Grafana HTTPS with certificate validation and confirm that all Prometheus targets return `up=1`.

---

# Security posture after remediation

The resulting security model is layered as follows:

1. **Proxmox host boundary**
   - Host firewall
   - LXC resource and device isolation
   - Host-controlled networking

2. **LXC confinement**
   - AppArmor remains enabled
   - Nesting remains disabled
   - Keyctl remains disabled
   - No broad mount or namespace permissions were granted

3. **Service identity**
   - Prometheus, Grafana, and PVE exporter run as dedicated unprivileged accounts

4. **Process restrictions**
   - No privilege escalation
   - Empty capability sets where supported
   - Restricted address families
   - SUID/SGID restrictions where supported
   - Runtime-only writable directory for the exporter

5. **Network exposure**
   - Prometheus and PVE exporter remain loopback-only
   - Grafana uses HTTPS
   - External access is controlled outside the service by the Proxmox firewall

6. **TLS trust**
   - The internal root CA public certificate is installed in the container trust store
   - Grafana can be validated without disabling certificate verification

---

# Rollback

## Remove service compatibility overrides

```bash
rm -f /etc/systemd/system/pve-exporter.service.d/security.conf
rm -f /etc/systemd/system/prometheus.service.d/lxc-compat.conf
rm -f /etc/systemd/system/grafana-server.service.d/lxc-compat.conf
systemctl daemon-reload
```

Do not restart the services until the reason for rollback is understood. Removing the overrides inside the same restrictive LXC will likely restore the original `226/NAMESPACE` failures.

## Unmask retired units

Only unmask units when the container design changes and the units are actually required:

```bash
systemctl unmask \
  dev-hugepages.mount \
  dev-mqueue.mount \
  run-lock.mount \
  tmp.mount \
  systemd-logind.service \
  systemd-networkd.service \
  systemd-networkd.socket
```

Do not enable `systemd-networkd` while `ifupdown` remains the active network manager.

## Remove the internal CA

```bash
rm -f /usr/local/share/ca-certificates/<homelab-root-ca>.crt
update-ca-certificates
```

This will cause internal HTTPS validation to fail unless another trusted chain is installed.

---

# Upgrade and maintenance notes

Package upgrades may modify vendor service units. Because the compatibility settings are stored as drop-ins under `/etc/systemd/system`, they should survive normal package upgrades.

After upgrades to any of the following, repeat the verification checklist:

- Proxmox host kernel
- LXC runtime
- AppArmor profiles
- systemd inside the container
- Prometheus
- Grafana
- Prometheus PVE Exporter

Pay particular attention to:

```text
status=226/NAMESPACE
Failed to set up mount namespacing
apparmor="DENIED"
```

A future systemd or LXC version may change which controls are compatible. Re-test controls individually before re-enabling them; do not assume old incompatibilities are permanent, and do not weaken the container globally to restore a single service.

---

# Outcome

The monitoring stack was restored without weakening the Proxmox LXC boundary. Prometheus, Grafana, and the PVE exporter run successfully; internal HTTPS validation works; monitoring targets are healthy; and the container reports no failed units.
