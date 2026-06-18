# Jellyfin Recovery: Missing NAS Mount Caused Reset-Looking Behavior

## Goal

Document the root cause and recovery steps for a Jellyfin issue where the server appeared partially reset, the media library metadata returned after restoring configuration, but playback initially failed.

## Summary

Jellyfin did not lose the media files. The media still existed on the NAS.

The issue happened because the NAS-backed CIFS mount was not active inside the Jellyfin LXC/container environment after a restart. Docker still started the Jellyfin container because the local mountpoint directory existed, but that directory was not actually backed by the NAS at the time.

As a result, Jellyfin started with an incomplete local directory structure instead of the real NAS-backed media and appdata paths.

## Root Cause

The root cause was:

```text
/mnt/media/jellyfin was not mounted to the NAS share
```

Expected state:

```text
//<NAS_IP>/<JELLYFIN_SHARE> -> /mnt/media/jellyfin
```

Broken state:

```text
/mnt/media/jellyfin existed locally, but was not mounted
```

Because the directory existed locally, Docker did not fail loudly. Instead, Jellyfin started using the local fallback directory.

## Expected Layout

When the NAS mount is active, the CT should see:

```text
/mnt/media/jellyfin/movies
/mnt/media/jellyfin/shows
/mnt/media/jellyfin/jellyfin-appdata
```

Inside the Jellyfin container, Docker maps this as:

```text
/media/movies
/media/shows
/media/jellyfin-appdata
```

## Broken Layout

When the NAS mount was missing, the CT only saw locally created appdata:

```text
/mnt/media/jellyfin/jellyfin-appdata
```

Inside the container, `/media` only contained appdata and did not contain the actual media folders:

```text
/media/jellyfin-appdata
```

Missing expected folders:

```text
/media/movies
/media/shows
```

## What Led to the Issue

During prior hardening and cleanup work, Jellyfin appdata was moved from local container storage to the NAS-backed path:

```text
/mnt/media/jellyfin/jellyfin-appdata/config
/mnt/media/jellyfin/jellyfin-appdata/cache
```

The Docker Compose volume mappings were updated to use NAS-backed appdata:

```yaml
volumes:
  - /mnt/media/jellyfin/jellyfin-appdata/config:/config
  - /mnt/media/jellyfin/jellyfin-appdata/cache:/cache
  - /mnt/media/jellyfin:/media
```

After a later restart, the CIFS mount did not come back before Docker/Jellyfin started.

This caused the following chain of events:

1. `/mnt/media/jellyfin` existed as a normal local directory.
2. Docker started Jellyfin successfully.
3. Jellyfin used the local fallback path instead of the NAS-backed path.
4. Jellyfin appeared reset because `/config` pointed to an incomplete or fresh appdata location.
5. After restoring the old configuration, users and libraries mostly returned.
6. Playback still failed because the restored Jellyfin database referenced media under `/media/shows/...`, but those files were not visible inside the container while the NAS mount was missing.

## Symptoms Observed

Jellyfin appeared reset or partially reset.

The old local configuration still existed:

```text
/opt/jellyfin/config/data/jellyfin.db
```

The old database was much larger than the fresh/reset database, which indicated the original configuration still existed and could likely be recovered.

Example comparison:

```text
Old/original config database: larger database file
Fresh/reset config database: smaller database file
```

After restoring the old configuration, the UI and library metadata mostly returned, but playback failed.

The playback failure was caused by Jellyfin/FFmpeg trying to read a media file path that did not exist inside the container:

```text
/media/shows/Example Show/Season 1/Example Episode.mkv
No such file or directory
```

FFmpeg exited with an error because the source file was not visible:

```text
FFmpeg exited with code 254
```

A direct file check inside the container also confirmed the file path was missing:

```bash
docker exec jellyfin sh -c '
ls -lh "/media/shows/Example Show/Season 1/Example Episode.mkv"
'
```

Result:

```text
No such file or directory
```

## Investigation Steps

### 1. Checked Docker Compose Volume Mappings

```bash
cd /opt/jellyfin

grep -nE 'config|cache|media|jellyfin-appdata' docker-compose.yml
```

Confirmed Jellyfin was configured to use:

```text
/mnt/media/jellyfin/jellyfin-appdata/config -> /config
/mnt/media/jellyfin/jellyfin-appdata/cache  -> /cache
/mnt/media/jellyfin                      -> /media
```

### 2. Checked Live Docker Mounts

```bash
docker inspect jellyfin --format '{{range .Mounts}}{{println .Source " -> " .Destination}}{{end}}'
```

This confirmed Docker was bind-mounting the expected host paths, but it did not prove that the host paths were backed by the NAS.

### 3. Compared Old Local Config vs NAS Appdata Config

```bash
find /opt/jellyfin/config -mindepth 1 -maxdepth 2 2>/dev/null | wc -l
find /opt/jellyfin/cache  -mindepth 1 -maxdepth 2 2>/dev/null | wc -l

find /mnt/media/jellyfin/jellyfin-appdata/config -mindepth 1 -maxdepth 2 2>/dev/null | wc -l
find /mnt/media/jellyfin/jellyfin-appdata/cache  -mindepth 1 -maxdepth 2 2>/dev/null | wc -l
```

The old local config had more data than the reset-looking config.

### 4. Checked Important Jellyfin Database Files

```bash
find /opt/jellyfin/config -maxdepth 4 -type f \
  \( -name 'jellyfin.db' -o -name 'system.xml' -o -name '*.db' \) \
  -printf '%TY-%Tm-%Td %TH:%TM  %10s bytes  %p\n' 2>/dev/null | sort

find /mnt/media/jellyfin/jellyfin-appdata/config -maxdepth 4 -type f \
  \( -name 'jellyfin.db' -o -name 'system.xml' -o -name '*.db' \) \
  -printf '%TY-%Tm-%Td %TH:%TM  %10s bytes  %p\n' 2>/dev/null | sort
```

This confirmed that the old database was the real/original Jellyfin configuration.

### 5. Checked Whether the NAS Mount Was Active

```bash
findmnt -n -o SOURCE,TARGET,FSTYPE /mnt/media/jellyfin || echo "not mounted"
```

Broken result:

```text
not mounted
```

This confirmed the main issue: the NAS share was not mounted.

### 6. Checked the Local Directory Tree

```bash
find /mnt/media -maxdepth 4 -type d | sort | sed -n '1,120p'
```

The output showed only local Jellyfin appdata and did not show the real NAS media folders.

### 7. Verified Media Was Missing Inside the Container

```bash
docker exec jellyfin sh -c '
find /media -maxdepth 3 -type d | sort | sed -n "1,80p"
'
```

Expected media folders were missing:

```text
/media/movies
/media/shows
```

## Recovery Steps

### 1. Stopped Jellyfin

```bash
cd /opt/jellyfin

docker compose stop jellyfin
```

### 2. Protected the Locally Restored Appdata

Before remounting the NAS share, the locally restored appdata was copied to a safe recovery location. This was necessary because mounting the NAS would hide whatever was currently stored under the local mountpoint.

```bash
cd /opt/jellyfin

stamp="$(date +%Y%m%d-%H%M%S)"

mkdir -p /opt/jellyfin/recovery

cp -a /mnt/media/jellyfin/jellyfin-appdata \
  /opt/jellyfin/recovery/jellyfin-appdata.local-restored-$stamp
```

Verified the protected database:

```bash
find /opt/jellyfin/recovery/jellyfin-appdata.local-restored-$stamp/config -maxdepth 4 -type f \
  \( -name 'jellyfin.db' -o -name 'system.xml' \) \
  -printf '%TY-%Tm-%Td %TH:%TM  %10s bytes  %p\n' 2>/dev/null | sort
```

### 3. Mounted the NAS Share

Checked the redacted fstab entry:

```bash
awk '
  $0 !~ /^[[:space:]]*#/ && $2=="/mnt/media/jellyfin" {
    gsub(/password=[^,[:space:]]+/, "password=REDACTED")
    gsub(/credentials=[^,[:space:]]+/, "credentials=REDACTED")
    print
  }
' /etc/fstab
```

Expected sanitized fstab structure:

```text
//<NAS_IP>/<JELLYFIN_SHARE>  /mnt/media/jellyfin  cifs  credentials=<CREDENTIAL_FILE>,uid=<UID>,gid=<GID>,iocharset=utf8,vers=3.0,_netdev,nofail,x-systemd.automount  0  0
```

Mounted the share:

```bash
mount /mnt/media/jellyfin
```

Verified the mount:

```bash
findmnt -n -o SOURCE,TARGET,FSTYPE /mnt/media/jellyfin
```

Expected result:

```text
//<NAS_IP>/<JELLYFIN_SHARE> /mnt/media/jellyfin cifs
```

### 4. Verified NAS Media Folders Returned

```bash
find /mnt/media/jellyfin -maxdepth 2 -type d | sort | sed -n '1,80p'
```

Expected folders:

```text
/mnt/media/jellyfin/movies
/mnt/media/jellyfin/shows
/mnt/media/jellyfin/jellyfin-appdata
```

### 5. Restored the Protected Config and Cache Onto the Real NAS Appdata

Selected the latest protected recovery folder:

```bash
latest_recovery="$(find /opt/jellyfin/recovery -maxdepth 1 -type d -name 'jellyfin-appdata.local-restored-*' | sort | tail -1)"
```

Backed up the current NAS appdata:

```bash
stamp="$(date +%Y%m%d-%H%M%S)"

mkdir -p /mnt/media/jellyfin/jellyfin-appdata/backups

mv /mnt/media/jellyfin/jellyfin-appdata/config \
   /mnt/media/jellyfin/jellyfin-appdata/backups/config.before-restore-$stamp

mv /mnt/media/jellyfin/jellyfin-appdata/cache \
   /mnt/media/jellyfin/jellyfin-appdata/backups/cache.before-restore-$stamp
```

Copied the protected restored config/cache to the real NAS appdata path:

```bash
cp -a "$latest_recovery/config" /mnt/media/jellyfin/jellyfin-appdata/config
cp -a "$latest_recovery/cache"  /mnt/media/jellyfin/jellyfin-appdata/cache
```

Verified the restored NAS database:

```bash
find /mnt/media/jellyfin/jellyfin-appdata/config -maxdepth 4 -type f \
  \( -name 'jellyfin.db' -o -name 'system.xml' \) \
  -printf '%TY-%Tm-%Td %TH:%TM  %10s bytes  %p\n' 2>/dev/null | sort
```

Expected result:

```text
/mnt/media/jellyfin/jellyfin-appdata/config/data/jellyfin.db
```

### 6. Started Jellyfin

```bash
cd /opt/jellyfin

docker compose up -d jellyfin
```

### 7. Verified Container Media Visibility

```bash
docker exec jellyfin sh -c '
ls -ld /media/shows /media/movies /media/jellyfin-appdata
'
```

Expected result:

```text
/media/shows
/media/movies
/media/jellyfin-appdata
```

### 8. Tested Playback

Playback was tested from the Jellyfin web UI and worked again.

## Final Working State

The NAS mount is active:

```text
//<NAS_IP>/<JELLYFIN_SHARE> -> /mnt/media/jellyfin
```

The CT sees the expected NAS-backed folders:

```text
/mnt/media/jellyfin/movies
/mnt/media/jellyfin/shows
/mnt/media/jellyfin/jellyfin-appdata
```

Docker maps the expected paths:

```text
/mnt/media/jellyfin/jellyfin-appdata/config -> /config
/mnt/media/jellyfin/jellyfin-appdata/cache  -> /cache
/mnt/media/jellyfin                      -> /media
```

The container sees the media paths again:

```text
/media/movies
/media/shows
/media/jellyfin-appdata
```

Playback works.

## Important Lesson

The media was never deleted.

The problem happened because Docker started while the NAS mount was missing. Since the mountpoint directory existed locally, Docker used that local directory instead of failing.

This created a misleading failure mode:

```text
Jellyfin looked reset, but the real issue was a missing NAS mount.
```

For NAS-backed Docker volumes, the NAS mount must be verified before starting containers that depend on it.

## Prevention / Follow-Up

Before starting Jellyfin, verify the NAS mount:

```bash
findmnt /mnt/media/jellyfin
```

Verify the expected media folders exist:

```bash
ls -ld /mnt/media/jellyfin/shows /mnt/media/jellyfin/movies /mnt/media/jellyfin/jellyfin-appdata
```

Verify the container sees the same paths:

```bash
docker exec jellyfin sh -c '
ls -ld /media/shows /media/movies /media/jellyfin-appdata
'
```

Recommended future improvement:

```text
Make Docker/Jellyfin wait for the NAS mount before starting.
```

This can be done by improving systemd mount dependencies, adding a pre-start mount check, or changing the service startup order so Jellyfin does not start unless `/mnt/media/jellyfin` is mounted.
