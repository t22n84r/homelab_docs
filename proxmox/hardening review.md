# Proxmox Host Hardening Record

## Scope
Hardening completed for a single-node Proxmox host.

---

## 1. Access Model

### Admin account
- Created a named admin account
- Used PAM-backed account model for:
  - host OS access
  - Proxmox GUI/API access
- Granted Proxmox `Administrator` role through group membership

### Sudo
- Installed `sudo`
- Added admin account to `sudo` group
- Verified standard sudo policy
- No dangerous custom `NOPASSWD: ALL` rules found

### Root
- Kept root as break-glass account only
- Do not use root for daily admin
- Do not rely on Proxmox UI user-disable for root as a real lockout control

### 2FA
- Set up TOTP path for admin account
- Recovery keys required
- WebAuthn/YubiKey deferred until proper hostname/TLS exists

---

## 2. SSH Hardening

### SSH key auth
- Generated SSH key on client
- Copied public key to host
- Verified key-based login works

### SSH policy
- Disabled password SSH
- Disabled root SSH login
- Kept key auth enabled
- Verified effective config with `sshd -T`
- Verified login behavior after changes

### Result
- Named admin user can SSH with key auth
- Root SSH disabled
- Password SSH disabled

---

## 3. GPU / Console Fallback Cleanup

### Goal
Ensure local console fallback is valid before tightening firewall rules.

### Findings
- Host GPU was using normal NVIDIA driver
- No active VM passthrough assignment remained
- Stale passthrough config still existed:
  - VFIO module load entries
  - stale NVIDIA/Nouveau blacklist file

### Cleanup
- Removed stale NVIDIA/Nouveau blacklist file
- Removed VFIO module lines from `/etc/modules`
- Rebuilt initramfs
- Rebooted

### Result
- GPU uses normal host driver
- `nvidia-smi` works
- VFIO no longer loaded
- Local console fallback valid from GPU/config side

---

## 4. Proxmox Firewall

### Enabled
- Enabled firewall at Datacenter level
- Enabled firewall at node level

### IP sets created
- `admin-desktop` = trusted admin source IPs
- `local-net` = LAN subnet
- `monitoring-hosts` = monitoring source IPs

### Important behavior discovered
- `management` is a special Proxmox firewall name
- Avoid using it for custom IP sets

### Host ports restricted
Rules implemented for:
- `22/tcp` SSH
- `8006/tcp` Proxmox UI/API
- `5900:5999/tcp` VNC console
- `3128/tcp` SPICE proxy
- `60000:60050/tcp` migration range
- `9100/tcp` node exporter

### Rule pattern
- `ACCEPT` trusted source sets first
- `DROP` broader LAN rules after

### Monitoring
- `9100/tcp` treated as Prometheus scrape endpoint
- Allowed only from monitoring source set
- Denied from rest of LAN

---

## 5. Logging / Journald

### Firewall logging
- Enabled node/global firewall logging
- Learned that per-rule `nolog` overrides broader logging expectations
- Practical policy:
  - do not log noisy ACCEPT rules
  - prefer logging DROP rules

### Journald limits
Created:

`/etc/systemd/journald.conf.d/99-limits.conf`

`ini
[Journal]
SystemMaxUse=1G
SystemMaxFileSize=50M
SystemMaxFiles=20
RateLimitIntervalSec=30s
RateLimitBurst=5000`

---

## 6. Disabled as unused
rpcbind.service
rpcbind.socket
nfs-blkmap.service
open-iscsi.service
corosync.service
pve-ha-crm.service
pve-ha-lrm.service
