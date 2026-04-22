# Raspberry Pi 4 DNS Server Hardening Record

## Overview

Dedicated Raspberry Pi 4 hardened for DNS infrastructure use.

Services:

- Pi-hole
- Unbound
- SSH
- UFW
- Fail2Ban

Role:

- Internal DNS resolver
- Network-wide ad blocking
- Secure administrative access

Status:

- Hardened
- Validated
- Backup-ready

---

# Security Controls Implemented

## Authentication

- Strong Pi-hole admin password configured
- Pi-hole 2FA enabled
- SSH key authentication enforced
- SSH password authentication disabled
- Root SSH login disabled

## SSH Hardening

Applied:

- Single authorized admin account
- Reduced brute-force tolerance
- Idle session timeout controls
- Configuration validation after changes

## Firewall

Default policy:

- deny incoming
- allow outgoing

Permitted only required services from trusted LAN:

- SSH
- DNS
- Pi-hole web UI

Removed obsolete temporary test ports.

## IPv6

Disabled system-wide.

## Service Reduction

Disabled / removed unused services:

- Avahi
- CUPS

## DNS Stack Hardening

Pi-hole exposed only for required DNS/web functions.

Unbound restricted to localhost only.

## Time Services

Disabled Pi-hole network NTP listener.

System time synchronization remains active.

## Intrusion Protection

Fail2Ban installed and configured.

## Maintenance

- sudo command logging enabled
- unattended security upgrades enabled

---

# Validation

## Internal Port Scan

Validated expected exposure only:

- SSH
- DNS
- Pi-hole HTTP UI
- Pi-hole HTTPS UI

All other scanned ports filtered.

## Listener Review

Validated only expected listeners remain active.

---

# Operational State

System considered:

- stable
- secure for home DNS role
- ready for imaging / backup clone

---

# Deferred Items

- centralized backup infrastructure
- granular sudo role separation
- historical Pi-hole data retention cleanup

---

# Summary

Converted Raspberry Pi into hardened dedicated DNS appliance with minimal attack surface and verified exposure.
