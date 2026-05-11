# Guacamole Remote Access — Public Notes

## Purpose

Deploy Apache Guacamole as a browser-based access hub for internal SSH/VNC/RDP connections through the remote access gateway.

## Stack Components

Docker Compose services:

```text
guacamole
guacd
guacamole-postgres
cloudflared
```

Guacamole roles:

- `guacamole`: web application.
- `guacd`: protocol proxy for VNC/RDP/SSH.
- `postgres`: Guacamole authentication/configuration database.
- `cloudflared`: Cloudflare Tunnel connector.

## Network Model

```text
Browser
  -> Cloudflare Access
  -> Cloudflare Tunnel
  -> guacamole container
  -> guacd container
  -> internal SSH/VNC/RDP targets
```

No Guacamole host port was intentionally published to the LAN/internet.

## Docker Compose Notes

Guacamole listens internally on:

```text
guacamole:8080
```

Guacd listens internally on:

```text
guacd:4822
```

PostgreSQL listens internally on:

```text
postgres:5432
```

## Database Initialization

Guacamole PostgreSQL schema is generated using:

```bash
docker run --rm guacamole/guacamole:<version> /opt/guacamole/bin/initdb.sh --postgresql
```

The generated file is mounted into Postgres init directory as:

```text
/docker-entrypoint-initdb.d/initdb.sql
```

## Issues Encountered

### `initdb.sql` Permission Denied

Initial database initialization failed because the Postgres container could not read the mounted SQL file.

Fix:

```text
Set initdb.sql mode to 0644
```

Then recreate the Postgres volume because initialization only runs on an empty database volume.

### Missing Guacamole Tables

Symptom:

```text
relation "guacamole_user" does not exist
```

Cause:

- Database schema was not initialized due to SQL file permission issue.

Fix:

- Correct file permission.
- Recreate database volume.

### Database Password Mismatch

Symptom:

```text
password authentication failed for user "guacamole_user"
```

Cause:

- Postgres volume was initialized with one password, then Compose/Vault used a different password later.

Fix options:

- Restore old password in Compose/Vault, or
- Reset/recreate the Guacamole Postgres volume on a fresh setup.

## Guacamole SSH Model

For SSH connections, use dedicated keys per target/system where possible.

Recommended model:

```text
Guacamole private key
  -> passphrase prompted at connection time
  -> server authorized_keys restricts key to gateway source IP
```

Do not reuse personal desktop keys for Guacamole automation/access.

## Guacamole VNC Model

VNC connections are routed through `guacd` to internal targets.

Validate from guacd network namespace:

```bash
docker run --rm --network container:guacd alpine sh -c 'apk add --no-cache busybox-extras >/dev/null && nc -vz -w 5 <target-ip> <port>'
```

## Status

- Guacamole app: working.
- PostgreSQL-backed auth/config: working after DB fixes.
- Cloudflare Access route: working.
- SSH access model: validated for server/Proxmox key usage.
- Desktop VNC testing belongs in separate desktop remote-access documentation.
