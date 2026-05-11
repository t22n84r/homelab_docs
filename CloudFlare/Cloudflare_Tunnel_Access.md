# Cloudflare Tunnel and Access — Public Notes

## Purpose

Use Cloudflare Tunnel and Cloudflare Access to provide browser-based access to internal services without exposing inbound ports on the home router.

## High-Level Flow

```text
Browser
  -> Cloudflare Access identity check
  -> Cloudflare Tunnel
  -> cloudflared connector on gateway VM
  -> internal service
```

## Security Model

- Cloudflare Tunnel creates outbound connections from the gateway VM to Cloudflare.
- No router port forwarding is required.
- Public hostnames exist, but access is gated by Cloudflare Access policies.
- Internal services should not be treated as safe just because they are behind a tunnel; they should still have their own authentication.

## Tunnel Connector

The tunnel connector runs as a Docker container:

```text
cloudflare/cloudflared:latest
```

The tunnel token is stored in Ansible Vault and rendered into Docker Compose at deployment time.

Do not store the token in:

```text
ansible-vault.pass
```

That file only unlocks Ansible Vault.

## Published Hostname Pattern

Guacamole route pattern:

```text
Hostname: guac.<domain>
Path: /guacamole
Service: http://guacamole:8080
```

Proxmox route pattern was discussed separately and belongs in hypervisor documentation.

## Access Application Pattern

Cloudflare Access application:

```text
Application type: Self-hosted
Domain: guac.<domain>
Policy: allow only approved email/account
Session duration: short
```

Policy behavior verified:

- Non-policy email did not receive OTP.
- Policy email received OTP and passed Cloudflare Access.

## Validation Commands

Check cloudflared logs:

```bash
sudo docker compose --project-directory /opt/remote-access-gateway logs --tail=120 cloudflared
```

Expected healthy indicators:

```text
Registered tunnel connection
Updated to new configuration
```

Test application reachability from Docker network:

```bash
sudo docker run --rm --network remote-access-net alpine sh -c 'apk add --no-cache wget >/dev/null && wget -S -O - http://guacamole:8080/guacamole 2>&1 | head -80'
```

Expected:

```text
HTTP/1.1 200
```

or Guacamole HTML output.

## Issues Encountered

### Route Path Required

The Guacamole public route worked after adding the `/guacamole` path.

Working browser URL:

```text
https://guac.<domain>/guacamole
```

### cloudflared Image Has No Shell

This failed because the `cloudflared` image is minimal:

```bash
docker exec cloudflared sh
```

Use a temporary container on the same Docker network instead.

## Do Not Put in Public Repo

- Real tunnel token.
- Real account IDs.
- Screenshots exposing Cloudflare account details.
- Any secret values.
