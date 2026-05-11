# Proxmox Access and Firewall — Public Notes

## Purpose

Document Proxmox access considerations related to the remote access gateway. This is separate from the gateway VM setup and belongs with hypervisor documentation.

## Scope

This document covers:

- Accessing Proxmox Web UI through Cloudflare Tunnel.
- Accessing Proxmox SSH from Guacamole/gateway.
- Proxmox firewall considerations for gateway-origin traffic.

## Proxmox Web UI Through Cloudflare Tunnel

Proposed route pattern:

```text
https://pve.<domain>
  -> Cloudflare Access
  -> Cloudflare Tunnel
  -> https://<proxmox-ip>:8006
```

Because Proxmox commonly uses a self-signed certificate, the tunnel route may need origin TLS verification disabled for that internal origin.

Cloudflare route pattern:

```text
Hostname: pve.<domain>
Service type: HTTPS
Service URL: <proxmox-ip>:8006
No TLS Verify: enabled
```

## Security Requirements

Do not expose Proxmox Web UI without strong layered controls.

Recommended layers:

```text
Cloudflare Access
  -> allow only approved email/account
  -> short session duration
  -> Proxmox login
  -> Proxmox 2FA
  -> Proxmox firewall source restrictions
```

## Proxmox SSH Through Guacamole

Guacamole can provide browser-based SSH access to Proxmox if the gateway is allowed to reach Proxmox SSH.

Recommended key model:

```text
Guacamole SSH private key
  -> passphrase prompted when connecting
  -> Proxmox authorized_keys restricts public key with source IP
```

Server-side key restriction pattern:

```text
from="<gateway-ip>" ssh-ed25519 <public-key> guacamole-to-server
```

## Firewall Rules

Required gateway access examples:

```text
Gateway VM -> Proxmox Web UI: tcp/8006
Gateway VM -> Proxmox SSH: tcp/22
```

Keep rules source-specific. Do not open Proxmox management ports broadly to the LAN if not needed.

## Status

- Gateway to Proxmox UI/SSH firewall allowance was completed.
- Proxmox Web UI route through Cloudflare was discussed as a separate hypervisor topic.
- This subject should stay out of the remote gateway VM setup documentation except as a cross-reference.
