# Raspberry Pi Bookworm Recovery Record

This document records the full recovery of a Raspberry Pi after an in-place Raspberry Pi OS upgrade left the system in a mixed Bullseye/Bookworm state.

## Table of Contents

1.  [Summary of Final State](https://www.google.com/search?q=%23summary-of-final-state)
2.  [Initial Issues](https://www.google.com/search?q=%23initial-issues)
3.  [Recovery Steps](https://www.google.com/search?q=%23recovery-steps)
      - [1. Fix Boot Mount Layout](https://www.google.com/search?q=%231-fix-boot-mount-layout)
      - [2. Repair Broken Package State](https://www.google.com/search?q=%232-repair-broken-package-state)
      - [3. Fix APT Legacy Key Warnings](https://www.google.com/search?q=%233-fix-apt-legacy-key-warnings)
      - [4. DNS Investigation](https://www.google.com/search?q=%234-investigate-temporary-dns-failure)
      - [5. Kernel Migration (Enable arm64 Path)](https://www.google.com/search?q=%235-enable-bookworm-kernel-package-path)
      - [6. Final Cleanup](https://www.google.com/search?q=%236-remove-old-kernel-track)
4.  [Quick Reference Command Summary](https://www.google.com/search?q=%23short-command-summary)

-----

## Summary of Final State

  * **Boot partition:** Fixed and mounted to `/boot/firmware`.
  * **Packages:** `raspi-firmware`, `raspi-config`, and `rc-gui` upgraded cleanly.
  * **APT:** Repository key warnings resolved using `/etc/apt/keyrings`.
  * **DNS:** Verified working (Pi-hole + Unbound).
  * **Kernel:** Migrated to the Bookworm path.
  * **Running Kernel:** `6.12.75+rpt-rpi-v8`

## Initial Issues

  * `raspi-firmware` failed because `/boot/firmware` was missing/not mounted.
  * The system used the legacy `/boot` mount layout.
  * `raspi-config` and `rc-gui` were "kept back" by APT.
  * APT warned about keys stored in the legacy `trusted.gpg` keyring.
  * System was stuck on the old kernel: `5.10.103-v8+`.

-----

## 1\. Fix Boot Mount Layout

> **Problem:** Bookworm expects the boot partition at `/boot/firmware`, but the system was still mounting it at `/boot`.

### Backup and Inspect

```bash
sudo cp /etc/fstab /etc/fstab.bak.$(date +%F-%H%M%S)
lsblk -f
cat /etc/fstab
```

### Update Mountpoint

```bash
sudo mkdir -p /boot/firmware
sudo umount /boot
```

### Update `/etc/fstab`

Change the boot partition line from `/boot` to `/boot/firmware`:

```text
PARTUUID=<boot-partition>  /boot/firmware  vfat  defaults  0 2
```

### Remount and Validate

```bash
sudo systemctl daemon-reload
sudo mount -a
findmnt /boot/firmware
```

-----

## 2\. Repair Broken Package State

### Repair Commands

```bash
sudo dpkg --configure -a
sudo apt --fix-broken install
sudo apt update
sudo apt full-upgrade
```

### Install Kept-back Packages

```bash
sudo apt install raspi-config rc-gui
sudo apt autoremove
```

-----

## 3\. Fix APT Legacy Key Warnings

### Export Keys to Keyrings

```bash
sudo mkdir -p /etc/apt/keyrings

# Export Raspbian Key
sudo gpg --no-default-keyring --keyring /etc/apt/trusted.gpg \
  --export <raspbian-key-fingerprint> \
  | sudo tee /etc/apt/keyrings/raspbian-archive-keyring.gpg >/dev/null

# Export Raspberry Pi Key
sudo gpg --no-default-keyring --keyring /etc/apt/trusted.gpg \
  --export <raspberrypi-key-fingerprint> \
  | sudo tee /etc/apt/keyrings/raspberrypi-archive-keyring.gpg >/dev/null

sudo chmod 0644 /etc/apt/keyrings/*.gpg
```

### Update Sources

Update `/etc/apt/sources.list` and `/etc/apt/sources.list.d/raspi.list` to include the `[signed-by=...]` parameter pointing to the new `.gpg` files.

-----

## 4\. Investigate Temporary DNS Failure

> **Symptom:** After reboot, external pings failed for \~30 seconds.
> **Result:** Confirmed as a startup timing issue. `pihole-FTL` and `unbound` eventually started correctly.

```bash
# Check local DNS health
sudo systemctl status pihole-FTL
dig @127.0.0.1 google.com
```

-----

## 5\. Enable Bookworm Kernel Package Path

### Add arm64 Architecture

Since the Pi was running `armhf` but capable of `arm64` (v8), the architecture was added to access the new kernel track.

```bash
sudo dpkg --add-architecture arm64
sudo apt update
```

### Install New Kernel

```bash
# Ensure config.txt has auto_initramfs=1 under [all]
sudo apt install linux-image-rpi-v8:arm64 raspi-firmware
sudo reboot
```

-----

## 6\. Remove Old Kernel Track

Once the system is verified to be running the new kernel (`uname -a`), remove the legacy packages:

```bash
sudo apt purge raspberrypi-kernel raspberrypi-bootloader
sudo apt autoremove --purge
```

-----

## Short Command Summary

```bash
# 1. Boot Layout
sudo mkdir -p /boot/firmware
sudo umount /boot
# Edit /etc/fstab here
sudo systemctl daemon-reload
sudo mount -a

# 2. Package Repair
sudo dpkg --configure -a
sudo apt --fix-broken install
sudo apt install raspi-config rc-gui

# 3. Kernel Migration
sudo dpkg --add-architecture arm64
sudo apt update
sudo apt install linux-image-rpi-v8:arm64 raspi-firmware
sudo reboot

# 4. Final Cleanup
sudo apt purge raspberrypi-kernel raspberrypi-bootloader
sudo apt autoremove --purge
```

> [\!NOTE]
> This recovery was performed in-place. While a fresh install is the "official" path, this system was successfully repaired and validated.
