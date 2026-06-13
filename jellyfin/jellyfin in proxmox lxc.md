# Jellyfin on Proxmox LXC with NVIDIA NVENC Hardware Transcoding

A complete guide to running Jellyfin in a Docker-based privileged LXC on Proxmox, with NVIDIA GPU passthrough for hardware transcoding.

---

## Overview

- **Platform:** Proxmox VE (Debian-based host)
- **Container:** Privileged LXC running Debian 12
- **Runtime:** Docker + Docker Compose inside the LXC
- **GPU:** NVIDIA GPU passed through to the LXC for NVENC/NVDEC hardware transcoding
- **Media sources:** Local drives on the Proxmox host + NAS over SMB/NFS

---

## Prerequisites

- Proxmox VE installed and running
- NVIDIA GPU installed in the server (not currently passed through to any active VM)
- A NAS or local drives with media
- Basic familiarity with Proxmox and Linux CLI

---

## Step 1 — Remove GPU from Any Existing VM

If the GPU is currently passed through to a VM, remove it first.

**Via Proxmox GUI:**
- Select the VM → Hardware → find the PCI Device entry → Remove

**Or via CLI on the Proxmox host:**
```bash
nano /etc/pve/qemu-server/<VMID>.conf
```
Remove the `hostpci` line referencing the GPU, save and exit. Do not start that VM again until the setup is complete.

---

## Step 2 — Install NVIDIA Drivers on the Proxmox Host

```bash
apt update && apt install -y pve-headers-$(uname -r) build-essential

# Add non-free repo
echo "deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware" \
  > /etc/apt/sources.list.d/non-free.list
apt update

# Install NVIDIA driver
apt install -y nvidia-driver firmware-misc-nonfree
```

Reboot the host:
```bash
reboot
```

After reboot, verify the driver loaded:
```bash
nvidia-smi
```

**Note the driver version number** — you will need to match it inside the LXC in Step 5.

---

## Step 3 — Ensure NVIDIA Kernel Modules Load on Boot

After the driver is installed, load the required modules:
```bash
modprobe nvidia
modprobe nvidia-modeset
modprobe nvidia-uvm
```

Persist them across reboots:
```bash
echo -e "nvidia\nnvidia-modeset\nnvidia-uvm" > /etc/modules-load.d/nvidia.conf
```

Verify all device nodes are present:
```bash
ls /dev/nvidia*
```

Expected output:
```
/dev/nvidia0
/dev/nvidiactl
/dev/nvidia-modeset
/dev/nvidia-uvm
/dev/nvidia-uvm-tools
```

> **Important:** If these device nodes do not exist when the LXC starts, the Jellyfin container will fail to start with an error about device nodes. The modules-load.d entry above ensures they are always created before any containers start.

---

## Step 4 — Create a Privileged LXC

**Via Proxmox GUI:**
- Create CT
- Download a **Debian 12** template if not already available
- **Uncheck "Unprivileged container"** — privileged is required for GPU passthrough
- Recommended resources: 4GB RAM, 4 CPU cores, 20GB disk
- Configure networking (DHCP or static IPv4)
- **Do not start the container yet**

**Critical — IPv6 networking fix:**
Proxmox injects network configuration into the container based on the GUI settings. If IPv6 is set to DHCP in the GUI but no DHCPv6 client is installed in the container, `networking.service` will fail on every boot, leaving the container with no IP address.

**Fix before starting the container:**
- Click the LXC → Network → edit `eth0`
- Set **IPv6** to **Static** with the address field left **blank**
- Click OK

This prevents Proxmox from writing `iface eth0 inet6 dhcp` into the container's `/etc/network/interfaces` on every start.

---

## Step 5 — Pass NVIDIA Device Nodes into the LXC

On the Proxmox host, get the major device numbers:
```bash
ls -la /dev/nvidia*
```
The number before the `:` in the output is the major number. For most systems this is `195` for nvidia devices and `510` for nvidia-uvm, but verify on your system.

Edit the LXC config:
```bash
nano /etc/pve/lxc/<CTID>.conf
```

Add the following lines:
```
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 510:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
```

Now start the LXC and enter it:
```bash
pct start <CTID>
pct enter <CTID>
```

---

## Step 6 — Install NVIDIA Drivers Inside the LXC (No Kernel Module)

Inside the LXC, install the **exact same driver version** as the host. The `--no-kernel-module` flag is critical — the kernel module runs on the host, not inside the container.

```bash
apt update && apt install -y wget build-essential

# Replace 5xx.xx.xx with your actual driver version from nvidia-smi on the host
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/5xx.xx.xx/NVIDIA-Linux-x86_64-5xx.xx.xx.run

chmod +x NVIDIA-Linux-x86_64-5xx.xx.xx.run
./NVIDIA-Linux-x86_64-5xx.xx.xx.run --no-kernel-module
```

Verify:
```bash
nvidia-smi
```

You should see the GPU listed from inside the container.

---

## Step 7 — Install Docker Inside the LXC

```bash
apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  > /etc/apt/sources.list.d/docker.list

apt update && apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Test:
```bash
docker run hello-world
```

---

## Step 8 — Deploy Jellyfin via Docker Compose

Create the working directory:
```bash
mkdir -p /opt/jellyfin && cd /opt/jellyfin
```

Create `docker-compose.yml`:
```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    network_mode: host
    environment:
      - JELLYFIN_PublishedServerUrl=http://<LXC_IP>:8096
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=all
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /mnt/local-media:/media/local
      - /mnt/nas-media:/media/nas
    devices:
      - /dev/nvidia0:/dev/nvidia0
      - /dev/nvidiactl:/dev/nvidiactl
      - /dev/nvidia-modeset:/dev/nvidia-modeset
      - /dev/nvidia-uvm:/dev/nvidia-uvm
      - /dev/nvidia-uvm-tools:/dev/nvidia-uvm-tools
    runtime: nvidia
```

Start Jellyfin:
```bash
docker compose up -d
docker ps
```

Access the web UI at `http://<LXC_IP>:8096` and complete the setup wizard. Skip media library setup for now if your mounts are not yet configured.

---

## Step 9 — Mount Media into the LXC

### Local drives on the Proxmox host

Add bind mounts to the LXC config on the Proxmox host:
```bash
# Edit /etc/pve/lxc/<CTID>.conf
mp0: /mnt/your-local-drive,mp=/mnt/local-media
```

### NAS over NFS

Inside the LXC:
```bash
apt install -y nfs-common
mkdir -p /mnt/nas-media
echo "192.168.x.x:/share/media  /mnt/nas-media  nfs  defaults  0  0" >> /etc/fstab
mount -a
```

### NAS over SMB/CIFS

```bash
apt install -y cifs-utils
echo "//192.168.x.x/media  /mnt/nas-media  cifs  credentials=/etc/nas-creds,uid=0,gid=0,x-systemd.automount  0  0" >> /etc/fstab
mount -a
```

> **Note:** Add `x-systemd.automount` to CIFS entries to prevent boot failures when the NAS is not yet reachable at mount time.

---

## Step 10 — Enable NVENC in Jellyfin

1. Log into the Jellyfin web UI
2. Navigate to: **Hamburger menu → Dashboard → Playback (left sidebar) → Transcoding**
3. Set **Hardware acceleration** to **NVENC**
4. Enable hardware decoding for all supported codecs:
   - ✅ H.264
   - ✅ H.265 / HEVC
   - ✅ VP9
   - ❌ AV1 — not supported on Pascal-era GPUs (GTX 10xx series)
5. Enable **Hardware encoding**
6. Enable **Tone mapping** if you have HDR content
7. Click **Save**

---

## Step 11 — Configure Automatic Library Scans

Jellyfin's filesystem change detection is unreliable inside Docker, especially with network shares. Set a scheduled scan instead:

**Dashboard → Scheduled Tasks → Scan Media Library** → set interval to every 15–30 minutes.

---
---

## Media Naming Conventions

Jellyfin uses TMDB/TVDB for metadata matching. Clean naming is essential.

**Movies:**
```
/media/movies/
  The Dark Knight (2008)/
    The Dark Knight (2008).mkv
```

**TV Shows:**
```
/media/shows/
  Breaking Bad/
    Season 01/
      Breaking Bad S01E01.mkv
      Breaking Bad S01E02.mkv
    Season 02/
      Breaking Bad S02E01.mkv
```

**Rules:**
- Show folder name must match the exact title on TMDB/TVDB
- Season folders must be named `Season 01`, `Season 02`, etc.
- Episode files must follow `S01E01` format
- Always include the year in parentheses for movies
- Specials go in `Season 00/`

---

## Troubleshooting

### Jellyfin container fails to start — device node error
```
error gathering device information while adding custom device "/dev/nvidia0": not a device node
```
The NVIDIA device nodes don't exist on the host. Load the modules:
```bash
modprobe nvidia && modprobe nvidia-modeset && modprobe nvidia-uvm
```
Then do a full stop/start of the LXC (not restart):
```bash
pct stop <CTID> && pct start <CTID>
```

### LXC has no IPv4 address / networking.service failed
Caused by Proxmox injecting `iface eth0 inet6 dhcp` into the container when no DHCPv6 client is installed. Fix in the Proxmox GUI: set the LXC network interface IPv6 to **Static** with a blank address. This prevents Proxmox from writing the problematic line on every container start.

### New media not appearing automatically
Inotify-based filesystem watching doesn't work reliably with Docker and network shares. Use scheduled scans instead: **Dashboard → Scheduled Tasks → Scan Media Library**.

### Jellyfin web UI unresponsive while TV continues streaming
Usually caused by the Jellyfin process getting overloaded during scans or metadata fetches, or by an unstable network interface inside the LXC. Check logs:
```bash
docker logs jellyfin --tail 100
```

---

## Boot Sequence

When everything is configured correctly, the full auto-start chain on Proxmox reboot is:

1. Proxmox host boots
2. `/etc/modules-load.d/nvidia.conf` loads NVIDIA kernel modules → device nodes created
3. LXC auto-starts → bind mounts attach device nodes into the container
4. Docker starts inside the LXC (systemd)
5. Jellyfin container starts automatically (`restart: unless-stopped`)

---

## Jellyfin Config and Cache Migration to Media Share

### Goal

Move Jellyfin’s persistent application data from local LXC storage to the mounted media share so that Jellyfin-related media state, metadata, configuration, and cache live alongside the media share instead of inside the container host’s local `/opt` path.

### Starting State

Jellyfin was running inside Docker in LXC `104`.

The existing Docker mounts were:

```text
/opt/jellyfin/config -> /config
/opt/jellyfin/cache  -> /cache
/mnt/media/jellyfin  -> /media
```

This meant:

```text
/config and /cache were stored locally inside the LXC:
/opt/jellyfin/config
/opt/jellyfin/cache

Media was stored on the mounted share:
/mnt/media/jellyfin
```

### Target State

Move Jellyfin config and cache to:

```text
/mnt/media/jellyfin/jellyfin-appdata/config
/mnt/media/jellyfin/jellyfin-appdata/cache
```

Final desired Docker mounts:

```text
/mnt/media/jellyfin/jellyfin-appdata/config -> /config
/mnt/media/jellyfin/jellyfin-appdata/cache  -> /cache
/mnt/media/jellyfin                         -> /media
```

### Steps Performed

1. Verified current Jellyfin Docker mounts from the Proxmox host:

```bash
sudo pct exec 104 -- docker inspect jellyfin --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

2. Confirmed Jellyfin was using local LXC paths for config and cache:

```text
/opt/jellyfin/config -> /config
/opt/jellyfin/cache -> /cache
/mnt/media/jellyfin -> /media
```

3. Created new appdata directories on the media share:

```bash
sudo pct exec 104 -- mkdir -p /mnt/media/jellyfin/jellyfin-appdata/config
sudo pct exec 104 -- mkdir -p /mnt/media/jellyfin/jellyfin-appdata/cache
```

4. Stopped the Jellyfin container before copying data:

```bash
sudo pct exec 104 -- docker stop jellyfin
```

5. Attempted to use `rsync`, but `rsync` was not installed inside LXC `104`.

6. Used `cp -a` instead to preserve file attributes while copying the existing Jellyfin data:

```bash
sudo pct exec 104 -- cp -a /opt/jellyfin/config/. /mnt/media/jellyfin/jellyfin-appdata/config/
sudo pct exec 104 -- cp -a /opt/jellyfin/cache/. /mnt/media/jellyfin/jellyfin-appdata/cache/
```

7. Verified the copied directories were populated:

```bash
sudo pct exec 104 -- sh -c 'find /mnt/media/jellyfin/jellyfin-appdata/config -maxdepth 1 -mindepth 1 | wc -l'
sudo pct exec 104 -- sh -c 'find /mnt/media/jellyfin/jellyfin-appdata/cache -maxdepth 1 -mindepth 1 | wc -l'
```

Observed results:

```text
config: 7 entries
cache: 5 entries
```

8. Updated the Jellyfin Docker Compose volume paths.

Original Compose volumes:

```yaml
volumes:
  - ./config:/config
  - ./cache:/cache
  - /mnt/media/jellyfin:/media
```

Updated Compose volumes:

```yaml
volumes:
  - /mnt/media/jellyfin/jellyfin-appdata/config:/config
  - /mnt/media/jellyfin/jellyfin-appdata/cache:/cache
  - /mnt/media/jellyfin:/media
```

9. Recreated the Jellyfin container using Docker Compose:

```bash
sudo pct exec 104 -- sh -c 'cd /opt/jellyfin && docker compose up -d'
```

10. Verified the new mounts:

```bash
sudo pct exec 104 -- docker inspect jellyfin --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Expected final output:

```text
/mnt/media/jellyfin/jellyfin-appdata/config -> /config
/mnt/media/jellyfin/jellyfin-appdata/cache -> /cache
/mnt/media/jellyfin -> /media
```

### Rollback Note

The old local directories were intentionally not deleted immediately:

```text
/opt/jellyfin/config
/opt/jellyfin/cache
```

They should be kept temporarily as a rollback backup until Jellyfin is confirmed working normally, including:

```text
users
libraries
metadata
plugins
watch history
media scanning
```

### Final Result

Jellyfin media, config, metadata, and cache are now intended to be stored under the media share path:

```text
/mnt/media/jellyfin/
```

with application data separated into:

```text
/mnt/media/jellyfin/jellyfin-appdata/
```

---

## Jellyfin LXC SMB Access Hardening

### Goal

Harden Jellyfin NAS access by replacing the broad SMB mount inside Jellyfin LXC `104` with a dedicated Jellyfin-only SMB share.

Previously, the LXC mounted the broad TrueNAS SMB share:

```text
//192.168.50.3/misc -> /mnt/media
```

Jellyfin only needed the Jellyfin media folder:

```text
/mnt/media/jellyfin
```

The Docker container mapped that folder into Jellyfin as:

```yaml
volumes:
  - /mnt/media/jellyfin:/media
```

The problem was that even though Docker only exposed `/media` to Jellyfin, the LXC itself still had access to the entire `misc` share.

The goal was to change the LXC mount to a dedicated Jellyfin-only SMB share:

```text
//192.168.50.3/jellyfin -> /mnt/media/jellyfin
```

---

### Why This Was Done

The old setup gave the Jellyfin LXC more NAS access than it needed.

Old access path:

```text
TrueNAS //192.168.50.3/misc
        ↓
LXC 104 /mnt/media
        ↓
Docker /media
```

This meant LXC `104` could access unrelated data inside the broader `misc` share, even though Jellyfin only required access to the Jellyfin media directory.

This violated least privilege.

If the Jellyfin LXC or container were compromised, the attacker could potentially browse or modify unrelated NAS data under the broad `misc` share.

The new design narrows the LXC’s NAS access to only the Jellyfin share:

```text
TrueNAS //192.168.50.3/jellyfin
        ↓
LXC 104 /mnt/media/jellyfin
        ↓
Docker /media
```

This reduces the blast radius while preserving Jellyfin functionality.

---

### Container-Side Changes

Entered Jellyfin LXC `104` from the Proxmox host:

```bash
sudo pct enter 104
```

Created the Jellyfin mount point inside the LXC:

```bash
mkdir -p /mnt/media/jellyfin
```

Created a dedicated SMB credentials directory and credentials file:

```bash
mkdir -p /root/.smb
micro /root/.smb/jellyfin.cred
```

The credentials file stores the dedicated Jellyfin SMB user credentials:

```ini
username=<REDACTED>
password=<REDACTED>
domain=WORKGROUP
```

Locked down the credentials file:

```bash
chown root:root /root/.smb/jellyfin.cred
chmod 600 /root/.smb/jellyfin.cred
```

---

### fstab Change

Edited the LXC `/etc/fstab`:

```bash
micro /etc/fstab
```

The old broad SMB mount was removed or commented out:

```fstab
# //192.168.50.3/misc  /mnt/media  cifs  credentials=...,uid=0,gid=0,iocharset=utf8,_netdev  0  0
```

Added the new Jellyfin-only SMB mount:

```fstab
//192.168.50.3/jellyfin  /mnt/media/jellyfin  cifs  credentials=/root/.smb/jellyfin.cred,uid=0,gid=0,iocharset=utf8,vers=3.0,_netdev,nofail,x-systemd.automount  0  0
```

The `x-systemd.automount` option was used so the SMB share mounts on demand when the path is accessed. This helps avoid boot-time network ordering issues.

Reloaded systemd and tested the mount:

```bash
systemctl daemon-reload
mount -a
findmnt -t cifs
```

Expected result:

```text
//192.168.50.3/jellyfin mounted at /mnt/media/jellyfin
```

The old broad mount should no longer appear:

```text
//192.168.50.3/misc
```

---

### Docker/Jellyfin Configuration

The Jellyfin Docker volume mapping did not need to change.

Existing Docker mapping:

```yaml
volumes:
  - /mnt/media/jellyfin:/media
```

From inside the Jellyfin container, the media path remains:

```text
/media
```

Only the backend SMB source changed.

Before:

```text
TrueNAS //192.168.50.3/misc
        ↓
LXC /mnt/media
        ↓
Docker /media
```

After:

```text
TrueNAS //192.168.50.3/jellyfin
        ↓
LXC /mnt/media/jellyfin
        ↓
Docker /media
```

---

### Validation

Confirmed the active CIFS mounts:

```bash
findmnt -t cifs
```

Confirmed the Jellyfin media path was accessible from inside the LXC:

```bash
ls -la /mnt/media/jellyfin
```

Tested write access because Jellyfin currently writes trickplay and metadata folders inside the media tree:

```bash
mkdir /mnt/media/jellyfin/smb_test
rmdir /mnt/media/jellyfin/smb_test
```

Restarted Jellyfin after confirming the mount:

```bash
docker restart jellyfin
```

Verified the Docker bind mount still points to the expected host path:

```bash
docker inspect jellyfin --format '{{json .Mounts}}'
```

Expected mapping:

```text
/mnt/media/jellyfin -> /media
```

---

### Final Result

Jellyfin LXC `104` no longer mounts the broad TrueNAS share:

```text
//192.168.50.3/misc
```

It now mounts only the dedicated Jellyfin SMB share:

```text
//192.168.50.3/jellyfin
```

Docker still exposes the media inside the Jellyfin container as:

```text
/media
```

This preserves Jellyfin functionality while reducing NAS exposure and aligning the LXC with least-privilege access.

