# Jellyfin Docker Image Update Notifications with Diun

## Goal

Set up a safe Docker image update notification process for Jellyfin without allowing automatic updates.

The purpose of this change was to support the new Jellyfin maintenance workflow:

1. Keep Jellyfin pinned to a known-good Docker image version.
2. Automatically check for newer Jellyfin image tags.
3. Send a notification when a relevant update is available.
4. Review the release notes before upgrading.
5. Manually update the pinned image only after review.

This avoids the risk of blindly running `latest` or allowing automatic container updates on a stateful service like Jellyfin.

---

## Why Diun Was Used

Diun was chosen because it is notification-focused.

Unlike Watchtower, the goal here was not to automatically pull and restart containers. Jellyfin has persistent configuration, cache, media mounts, and possible database/plugin compatibility concerns, so updates should be reviewed before being applied.

Diun fits this workflow because it:

* Detects Docker image updates.
* Supports scheduled checks.
* Can watch only specifically labeled containers.
* Can notify through ntfy.
* Does not update containers by itself.
* Works well with pinned image tags and manual review.

---

## Design Decision

The update process is intentionally split into two parts:

```text
Diun = detect and notify
Human/ChatGPT review = release review and verdict
Docker Compose = manual controlled update
```

The final process is:

```text
Diun checks daily
↓
ntfy sends detailed update notification
↓
Notification is pasted into ChatGPT for release review
↓
Verdict is given: update soon / safe to schedule / review carefully / skip
↓
Jellyfin image tag is manually updated in docker-compose.yml
↓
Container is recreated only after approval
```

This keeps automation where it is useful while avoiding uncontrolled production changes.

---

## Environment

Jellyfin runs inside a Docker container within Proxmox LXC CT 104.

Current setup at time of implementation:

```text
Host/container: Proxmox LXC CT 104
Application: Jellyfin
Docker image: jellyfin/jellyfin:10.11.6
Running version: Jellyfin.Server 10.11.6.0
Update model: pinned image, manual updates only
Notification tool: Diun
Notification target: ntfy
```

---

## What Was Implemented

A new Diun Docker Compose stack was created under:

```text
/opt/diun
```

Diun data is stored under:

```text
/opt/diun/data
```

The Diun environment file is stored at:

```text
/opt/diun/.env
```

The `.env` file contains the private ntfy topic and should not be committed to GitHub.

The Diun Compose file is stored at:

```text
/opt/diun/docker-compose.yml
```

---

## Diun Configuration Summary

Diun was configured with the following behavior:

```text
Schedule: daily at 9:00 AM
Run on startup: enabled
First-check notifications: disabled
Digest comparison: enabled
Docker provider: enabled
Watch stopped containers: disabled
Watch by default: disabled
Notification method: ntfy
```

Important behavior:

```text
DIUN_PROVIDERS_DOCKER_WATCHBYDEFAULT=false
```

This means Diun ignores containers unless they are explicitly labeled.

---

## Security Notes

The ntfy topic is treated like a secret.

Do not commit this file:

```text
/opt/diun/.env
```

Do not commit the real ntfy topic value.

The Docker socket is mounted read-only into the Diun container:

```yaml
- /var/run/docker.sock:/var/run/docker.sock:ro
```

This still gives Diun visibility into Docker metadata, but avoids giving it write access through the socket mount.

---

## Diun Docker Compose Stack

Diun was deployed as a separate Docker Compose stack:

```yaml
name: diun

services:
  diun:
    image: crazymax/diun:4.33.0
    container_name: diun
    hostname: jellyfin-ct104-diun
    command: serve

    env_file:
      - .env

    environment:
      TZ: America/New_York
      LOG_LEVEL: info
      LOG_JSON: "false"

      DIUN_WATCH_WORKERS: "10"
      DIUN_WATCH_SCHEDULE: "0 9 * * *"
      DIUN_WATCH_JITTER: "30s"
      DIUN_WATCH_FIRSTCHECKNOTIF: "false"
      DIUN_WATCH_RUNONSTARTUP: "true"
      DIUN_WATCH_COMPAREDIGEST: "true"

      DIUN_PROVIDERS_DOCKER: "true"
      DIUN_PROVIDERS_DOCKER_WATCHBYDEFAULT: "false"
      DIUN_PROVIDERS_DOCKER_WATCHSTOPPED: "false"

      DIUN_NOTIF_NTFY_ENDPOINT: "https://ntfy.sh"
      DIUN_NOTIF_NTFY_PRIORITY: "3"
      DIUN_NOTIF_NTFY_TAGS: "whale,package"
      DIUN_NOTIF_NTFY_TIMEOUT: "10s"
      DIUN_NOTIF_NTFY_CLICK: "{{ .Entry.Metadata.release_notes }}"

      DIUN_NOTIF_NTFY_TEMPLATETITLE: "Docker update: {{ .Entry.Metadata.app }} {{ .Entry.Image.Tag }}"

      DIUN_NOTIF_NTFY_TEMPLATEBODY: |-
        Docker image update detected.

        App: {{ .Entry.Metadata.app }}
        Host: {{ .Meta.Hostname }}
        Container: {{ .Entry.Metadata.container_name }}
        Compose path: {{ .Entry.Metadata.compose_path }}
        Stack: {{ .Entry.Metadata.stack }}
        Update policy: notification only; manual review before update.

        Current pinned version: {{ .Entry.Metadata.current_version }}
        Available image: {{ .Entry.Image }}
        Available tag: {{ .Entry.Image.Tag }}
        Status: {{ .Entry.Status }}
        Provider: {{ .Entry.Provider }}
        Registry: {{ .Entry.Image.Domain }}
        Digest: {{ .Entry.Manifest.Digest }}
        Hub link: {{ .Entry.Image.HubLink }}
        Release notes: {{ .Entry.Metadata.release_notes }}

        Local context:
        - Jellyfin runs in Docker inside Proxmox LXC CT 104.
        - No public exposure.
        - No GPU/transcoding currently; 1080p only.
        - Media is mounted from NAS.
        - Desired behavior: notify only, do not auto-update.

        Paste this into ChatGPT:

        Please review this Jellyfin Docker image update and give me a verdict:
        UPDATE SOON, SAFE TO SCHEDULE, REVIEW CAREFULLY, or SKIP FOR NOW.

        Check release notes, security fixes, breaking changes, database migrations,
        Docker image changes, plugin compatibility, known regressions, and whether
        this matters for my setup.

    volumes:
      - ./data:/data
      - /var/run/docker.sock:/var/run/docker.sock:ro

    healthcheck:
      test: ["CMD", "diun", "healthcheck"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 60s

    restart: unless-stopped
```

---

## Jellyfin Labels Added

Jellyfin was configured with Diun labels so Diun can monitor it.

The Jellyfin image remains pinned:

```yaml
image: jellyfin/jellyfin:10.11.6
```

The following labels were added to the Jellyfin service:

```yaml
labels:
  - "diun.enable=true"
  - "diun.watch_repo=true"
  - "diun.sort_tags=semver"
  - "diun.max_tags=20"
  - "diun.include_tags=^10\\.[0-9]+\\.[0-9]+$"
  - "diun.notify_on=new;update"
  - "diun.metadata.app=Jellyfin"
  - "diun.metadata.container_name=jellyfin"
  - "diun.metadata.stack=jellyfin"
  - "diun.metadata.compose_path=/opt/jellyfin/docker-compose.yml"
  - "diun.metadata.current_version=10.11.6"
  - "diun.metadata.release_notes=https://github.com/jellyfin/jellyfin/releases"
  - "diun.metadata.update_policy=manual-review"
```

The `diun.watch_repo=true` label allows Diun to watch for newer repository tags instead of only checking the exact currently pinned tag.

The `diun.include_tags` regex limits notifications to Jellyfin stable `10.x.x` tags.

---

## Commands Used

Created the Diun directory:

```bash
mkdir -p /opt/diun/data
chmod 750 /opt/diun
chmod 750 /opt/diun/data
```

Created a private `.env` file for the ntfy topic:

```bash
python3 - <<'PY' > /opt/diun/.env
import secrets
print("DIUN_NOTIF_NTFY_TOPIC=diun-" + secrets.token_urlsafe(24))
PY

chmod 600 /opt/diun/.env
```

Validated the Diun Compose file:

```bash
cd /opt/diun
docker compose config >/dev/null && echo "diun compose file is valid"
```

Started Diun:

```bash
cd /opt/diun
docker compose up -d
```

Confirmed Diun was running:

```bash
docker compose ps
```

Tested ntfy notification:

```bash
docker exec diun diun notif test
```

Backed up the Jellyfin Compose file before adding labels:

```bash
cd /opt/jellyfin
cp docker-compose.yml docker-compose.yml.bak-pre-diun-labels
```

Confirmed the running Jellyfin version:

```bash
docker exec jellyfin /jellyfin/jellyfin --version
```

Validated the Jellyfin Compose file after adding labels:

```bash
cd /opt/jellyfin
docker compose config >/dev/null && echo "jellyfin compose file is valid"
```

Recreated Jellyfin so Docker would apply the new labels:

```bash
cd /opt/jellyfin
docker compose up -d
```

Verified Jellyfin remained pinned and healthy:

```bash
docker ps --filter name=jellyfin
docker exec jellyfin /jellyfin/jellyfin --version
```

Verified Diun labels were applied:

```bash
docker inspect jellyfin --format '
diun.enable={{ index .Config.Labels "diun.enable" }}
app={{ index .Config.Labels "diun.metadata.app" }}
current_version={{ index .Config.Labels "diun.metadata.current_version" }}
release_notes={{ index .Config.Labels "diun.metadata.release_notes" }}
'
```

Forced Diun to scan:

```bash
cd /opt/diun
docker compose restart diun
docker compose logs --tail=120 diun
```

---

## Validation Results

Diun container status:

```text
diun      crazymax/diun:4.33.0   "diun serve"   diun   Up   healthy
```

Jellyfin remained pinned:

```text
IMAGE                       NAMES
jellyfin/jellyfin:10.11.6   jellyfin
```

Jellyfin running version:

```text
Jellyfin.Server 10.11.6.0
```

Diun labels confirmed:

```text
diun.enable=true
app=Jellyfin
current_version=10.11.6
release_notes=https://github.com/jellyfin/jellyfin/releases
```

Diun first detected matching Jellyfin tags and created its baseline:

```text
Jobs completed added=20 failed=0 skipped=0 unchanged=0 updated=0
```

After restarting Diun again, the same tags were recognized as unchanged:

```text
Jobs completed added=0 failed=0 skipped=0 unchanged=20 updated=0
```

This confirms Diun is not repeatedly alerting on old tags.

---

## Operational Workflow

Daily process:

```text
Diun checks for matching Jellyfin image tags at 9:00 AM.
If a new matching tag appears, Diun sends an ntfy notification.
The notification contains enough context to paste into ChatGPT for review.
The update is only applied manually after review.
```

Review categories:

```text
UPDATE SOON       = security fix or important stability fix
SAFE TO SCHEDULE  = normal patch release
REVIEW CAREFULLY  = possible breaking change, migration, plugin issue, or known regression
SKIP FOR NOW      = bad release, irrelevant update, or risk outweighs benefit
```

---

## Manual Jellyfin Update Procedure

When an update is approved:

```bash
cd /opt/jellyfin
cp docker-compose.yml docker-compose.yml.bak-$(date +%F)-pre-jellyfin-update
micro docker-compose.yml
```

Update both the image tag and metadata version.

Example:

```yaml
image: jellyfin/jellyfin:10.11.7
```

Also update:

```yaml
- "diun.metadata.current_version=10.11.7"
```

Then apply the update:

```bash
docker compose pull jellyfin
docker compose up -d jellyfin
docker exec jellyfin /jellyfin/jellyfin --version
docker compose logs --tail=100 jellyfin
```

---

## Current Final State

```text
Diun is installed and healthy.
ntfy notification test succeeded.
Jellyfin is labeled for Diun monitoring.
Jellyfin remains pinned to 10.11.6.
Diun checks daily at 9:00 AM.
Old tags have been baselined and are not repeatedly alerting.
Future Jellyfin updates will trigger review-ready notifications.
```

---

## Files Not Safe for Public GitHub

Do not commit:

```text
/opt/diun/.env
```

Do not publish:

```text
Real ntfy topic value
Any private notification URLs
Any secrets or tokens
```

The documentation above is safe to publish as long as the real ntfy topic is not included.
