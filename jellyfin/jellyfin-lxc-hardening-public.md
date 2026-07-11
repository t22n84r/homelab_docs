# Jellyfin LXC Hardening Record

> Public-safe documentation. Internal IPs, usernames, secrets, and unresolved security gaps are intentionally omitted or generalized.

## Purpose

This document records the practical hardening performed on a Jellyfin deployment running in Docker inside a Proxmox LXC container.

The goal was to reduce unnecessary exposure while preserving normal Jellyfin functionality, including LAN access, metadata handling, subtitle handling, and playback.

## Architecture

```text
Proxmox Host
└── LXC Container
    ├── Docker Engine
    │   ├── Jellyfin container
    │   └── DIUN update-notification container
    └── CIFS/SMB mount to a dedicated Jellyfin media share
```

## Scope

This hardening pass focused on the Jellyfin LXC and Docker runtime.

In scope:

- LXC service exposure
- Docker startup safety
- Jellyfin Docker Compose cleanup
- NAS mount narrowing
- Firewall containment
- IPv6 disablement
- Update/backup workflow
- SMB credential file permissions
- Jellyfin application-level user/plugin review

Out of scope:

- NAS-side data protection
- Broader home network segmentation
- Full container runtime redesign

## Starting Problem

The original Jellyfin setup had a few practical risks:

- The LXC previously had access to a broader NAS share than necessary.
- Jellyfin could start even if the NAS mount was missing.
- SSH/socket activation was present in the LXC.
- Old GPU/NVIDIA configuration remained even though hardware transcoding was no longer used.
- The container had unnecessary IPv6 exposure.
- The firewall rules did not yet enforce tight inbound/outbound boundaries.
- Update workflow needed to be safer and more deliberate.

## Major Changes Completed

### 1. NAS Share Narrowing

The LXC was changed from using a broad NAS share to a dedicated Jellyfin-only SMB share.

Result:

```text
Before: broad NAS share exposed to the LXC
After: only the dedicated Jellyfin share is mounted
```

This limits the blast radius if Jellyfin or the LXC is compromised.

### 2. Docker Compose Cleanup

The Jellyfin Compose file was simplified and cleaned.

Key changes:

- Removed unused GPU/NVIDIA runtime configuration.
- Kept Jellyfin on host networking for LAN discovery and simple access.
- Pinned Jellyfin to a specific version instead of using a floating tag.
- Kept `/config` and `/cache` on persistent appdata storage.
- Kept media mounted read-write because sidecar metadata/trickplay/subtitle files are intentionally stored beside media.

Generalized Compose layout:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:<pinned-version>
    container_name: jellyfin
    restart: unless-stopped
    network_mode: host
    volumes:
      - <appdata-path>/config:/config
      - <appdata-path>/cache:/cache
      - <media-path>:/media
```

### 3. Update Notification Workflow

DIUN was added as an update notification tool.

Purpose:

- Watch for Jellyfin image updates.
- Notify only.
- Avoid automatic upgrades.
- Require manual review before updating Jellyfin.

This keeps the environment stable while still surfacing update availability.

The notification workflow was also updated to remind the operator to run the backup-before-upgrade script before applying any Jellyfin image change.

### 4. Backup-Before-Upgrade Script

A backup script was added before Jellyfin upgrades.

The script backs up:

- Jellyfin config directory
- Docker Compose file

It also applies retention to keep only the newest backups.

General behavior:

```text
Run backup script before changing Jellyfin image version.
Verify backup exists.
Then perform the manual upgrade.
```

### 5. Docker Mount Guard

A systemd drop-in was added for `docker.service` so Docker will not start unless the Jellyfin NAS mount is actually present.

This prevents the failure mode:

```text
NAS mount missing
→ Docker starts Jellyfin
→ Jellyfin sees an empty config path
→ Jellyfin initializes like a fresh server
```

Implemented guard:

```ini
[Unit]
Requires=<jellyfin-mount-unit>.mount
After=<jellyfin-mount-unit>.mount

[Service]
ExecStartPre=/usr/bin/mountpoint -q <jellyfin-mount-path>
```

Verified:

- Docker restarted successfully.
- Mount guard ran successfully.
- Jellyfin came back healthy.
- DIUN came back healthy.
- Jellyfin web UI and playback still worked.

### 6. SSH Disabled

SSH was disabled inside the LXC.

Important note: both the service and socket activation were checked. The SSH service alone was not enough because the socket could still listen on port `22`.

Final state:

```text
ssh.service = disabled/inactive
ssh.socket  = disabled/inactive
port 22     = not listening
```

### 7. Postfix Disabled

Postfix was disabled because the LXC did not need to run a local mail listener.

Final state:

```text
postfix = disabled/inactive
mail port = not listening
```

### 8. IPv6 Disabled

IPv6 was disabled inside the LXC because the home network does not use it for this service.

Verified:

```text
No IPv6 addresses
No IPv6 listeners
```

### 9. GPU/NVIDIA Cleanup

Old GPU passthrough configuration was removed from the LXC config.

NVIDIA-related services were disabled because hardware transcoding was not currently being used.

Packages were left installed for possible future reuse.

### 9a. Docker-in-LXC Runtime Check

The Docker-in-LXC runtime behavior was reviewed during hardening. A change that would have broken Docker/Jellyfin startup was not kept. The final state preserves the minimum settings needed for Docker to function correctly in this LXC.

### 9b. Jellyfin Application-Level Review

Jellyfin application permissions were reviewed separately from host/container hardening.

Completed at the application level:

- Non-admin user permissions were restricted.
- Server-management access was kept limited to the admin account.
- Deletion and unnecessary remote-control/download/live-TV style permissions were disabled where not needed.
- Login-screen visibility was reduced for relevant accounts.
- Installed plugins were reviewed and kept where required for metadata/subtitle functionality.

### 10. Proxmox Firewall Containment

The CT firewall was enabled on the LXC network interface.

Inbound policy:

```text
Default inbound: DROP
Allow Jellyfin web UI from LAN
Allow Jellyfin LAN discovery from LAN
Allow DHCP replies
```

Outbound policy:

```text
Default outbound: ACCEPT
Allow required local services first
Then block unnecessary outbound access to the LAN
```

This allows Jellyfin to keep working while limiting LAN pivot potential from inside the container.

Verified:

- Jellyfin web UI works.
- Playback works.
- NAS SMB access works.
- DNS works.
- External HTTPS works.
- Router/Proxmox management access from the CT is blocked or unavailable.
- Proxmox firewall is enabled/running.
- CT NIC has `firewall=1`.

### 11. SMB Credential Permissions

The SMB credential file and parent directory were locked down.

Final state:

```text
credential file: 600 root:root
credential dir:  700 root:root
```

No secret values are documented here.

## Final Expected Listeners

After cleanup, the only expected listeners inside the LXC are:

```text
tcp/8096  Jellyfin web UI
udp/7359  Jellyfin LAN discovery
udp/68    DHCP client
```

SSH should not be listening.

## Final Functional Verification

After all changes:

```text
Jellyfin web UI loads
Login works
Playback works
Docker starts cleanly
Jellyfin container is healthy
DIUN container is healthy
NAS mount is active
Firewall is active
IPv6 is disabled
SSH is not listening
```

## Operational Notes

### Before upgrading Jellyfin

```text
1. Run the backup-before-upgrade script.
2. Confirm backup files were created.
3. Review Jellyfin release notes.
4. Update the pinned image version in Compose.
5. Pull and restart Jellyfin.
6. Verify web UI, playback, metadata, and logs.
```

### If Jellyfin appears reset again

Check the NAS mount first:

```bash
findmnt <jellyfin-mount-path>
```

Then check Docker service status:

```bash
systemctl status docker --no-pager
```

The Docker mount guard should prevent Docker from starting Jellyfin when the mount is missing.

## Final Status

```text
Status: Hardening pass complete
Result: Practical containment achieved without breaking Jellyfin functionality
```
