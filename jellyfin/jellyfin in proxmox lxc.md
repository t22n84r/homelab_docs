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
