# Remote Access Gateway Hardening - Public Documentation

## Overview

This document summarizes the hardening work completed for a dedicated remote access gateway VM. The gateway provides controlled browser-based access to internal systems through Cloudflare Tunnel, Cloudflare Access, and Apache Guacamole.

This public version intentionally avoids private values such as IP addresses, domains, usernames, token values, passwords, private keys, and exact internal network layout.

## Goals

- Reduce exposed services on the gateway VM.
- Keep administrative access available through SSH and Ansible.
- Prevent direct LAN exposure of Guacamole, PostgreSQL, and guacd.
- Restrict sensitive runtime files to root-only access.
- Keep Docker services internal unless intentionally routed through Cloudflare Tunnel.
- Validate that hardening changes survive updates and reboot.

## Scope

Included:

- SSH hardening
- Ansible access validation
- Fail2Ban validation
- Listening port review
- Docker exposure review
- Docker network review
- Docker socket review
- Runtime secret-file permission hardening
- Cloudflared container update
- Container restart-policy validation
- Unattended upgrades validation
- Kernel update and reboot validation
- Secret rotation for previously exposed runtime secrets

Not included:

- Cloudflare Access policy design details
- Guacamole application configuration details
- Desktop remote-access testing
- Proxmox host hardening
- Backup infrastructure design

## Threat Model

The gateway VM is a high-value internal access point because it can reach selected internal systems and runs services that broker remote access. If compromised, it could become a pivot point into the internal environment.

Main risks addressed:

- SSH brute-force or password login risk
- Accidental exposure of Guacamole/PostgreSQL/guacd to the LAN
- Overly broad Docker access
- Runtime secrets readable by non-root users
- Stale containers or packages
- Hardening changes breaking automation access

## SSH Hardening

SSH was hardened using a drop-in configuration under `/etc/ssh/sshd_config.d/`.

Final posture:

```sshconfig
SyslogFacility AUTH
LogLevel INFO

LoginGraceTime 30
PermitRootLogin no
StrictModes yes

MaxAuthTries 3
MaxSessions 2
MaxStartups 3:30:10

PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
AuthenticationMethods publickey

HostbasedAuthentication no
IgnoreRhosts yes

PasswordAuthentication no
PermitEmptyPasswords no
KbdInteractiveAuthentication no

AllowAgentForwarding no
AllowTcpForwarding no
AllowStreamLocalForwarding no
GatewayPorts no
X11Forwarding no
PermitTTY yes
PrintLastLog yes
PermitUserEnvironment no
ClientAliveInterval 300
ClientAliveCountMax 2
PermitTunnel no
StreamLocalBindUnlink no
UseDNS no

AllowUsers <admin-user> <automation-user>
```

Validation commands:

```bash
sudo sshd -t
systemctl status ssh --no-pager
```

Result:

- SSH remained active after restart and reboot.
- Admin key login worked.
- Automation key login worked.
- Password SSH was disabled.

## Ansible Access Validation

After SSH hardening and reboot, Ansible access was validated from the automation VM.

Validation checks:

```bash
ansible remote_access_gateway -m ping --vault-password-file <vault-password-file>
ansible remote_access_gateway -m command -a "whoami" -b --vault-password-file <vault-password-file>
```

Result:

- Ansible ping succeeded.
- Ansible privilege escalation succeeded and returned `root`.

## Fail2Ban Validation

Fail2Ban was installed and active with an SSH jail.

Validation commands:

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

Result:

- `sshd` jail was active.
- No failed or banned IPs were present during validation.

## Listening Port Review

Listening ports were reviewed with:

```bash
sudo ss -tulpn
```

Expected exposure:

- SSH on TCP/22
- Local resolver stubs on loopback
- DHCP client traffic

Result:

- No Guacamole web port was published on the host.
- No PostgreSQL port was published on the host.
- No guacd port was published on the host.
- No Docker TCP API was exposed.

## Docker Published Port Review

Docker-published ports were reviewed with:

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
```

Result:

- Guacamole showed only its internal container port.
- PostgreSQL showed only its internal container port.
- guacd showed only its internal container port.
- cloudflared had no published port.
- No `0.0.0.0:PORT->...` mappings existed for Guacamole, PostgreSQL, or guacd.

## Docker Network Review

Docker networks were reviewed with:

```bash
sudo docker network ls
sudo docker network inspect <remote-access-network>
```

Expected containers on the remote-access network:

- `cloudflared`
- `guacamole`
- `guacamole-postgres`
- `guacd`

Result:

- Only the expected containers were attached to the private remote-access Docker network.

## Docker Socket Review

Docker socket exposure was checked with:

```bash
sudo ss -xlpn | grep docker
sudo ls -l /var/run/docker.sock
getent group docker
```

Result:

- Docker was exposed only through Unix sockets.
- No TCP Docker API listener was present.
- No regular users were members of the `docker` group.

## Runtime Secret File Permissions

The rendered Docker Compose file was confirmed to contain runtime secrets needed by the stack. Its permissions were tightened to root-only.

Final posture:

```text
/opt/remote-access-gateway/                 750 root root
docker-compose.yml                          600 root root
initdb.sql                                  644 root root
```

The Ansible playbook was patched so future renders keep `docker-compose.yml` at `0600`.

Important practice:

- Do not print rendered secret values to terminal during troubleshooting.
- Use permission checks, file existence checks, masked output, or hashes instead.

## Guacamole SSH Key Storage Review

Dedicated Guacamole SSH keys were stored under a root-only directory.

Expected posture:

```text
/root/guacamole-ssh-keys/                   700 root root
private keys                                600 root root
public keys                                 644 root root
known_hosts                                 644 or 600 root root
```

Result:

- Key directory was root-only.
- Private keys were root-only.
- Public keys and known_hosts had acceptable permissions.

## Cloudflared Update

Cloudflared was updated by pulling the current container image and recreating the container.

Validation checks:

```bash
sudo docker compose --project-directory /opt/remote-access-gateway pull cloudflared
sudo docker compose --project-directory /opt/remote-access-gateway up -d cloudflared
sudo docker compose --project-directory /opt/remote-access-gateway logs --tail=40 cloudflared
```

Result:

- Cloudflared updated successfully.
- Tunnel registered successfully after update.

## Container Restart Policies

Restart policy was checked with:

```bash
sudo docker inspect --format='{{.Name}} {{.HostConfig.RestartPolicy.Name}}' guacamole guacamole-postgres guacd cloudflared
```

Result:

- All remote-access containers used `unless-stopped`.

## Package Update and Reboot Validation

Unattended upgrades were active and package update history showed successful unattended upgrades.

Validation commands:

```bash
systemctl status unattended-upgrades --no-pager
sudo tail -80 /var/log/unattended-upgrades/unattended-upgrades.log
sudo tail -80 /var/log/apt/history.log
```

A manual package update installed a newer kernel and required a reboot. After reboot, the active kernel matched the new installed kernel.

Post-reboot checks:

```bash
uname -r
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
systemctl status ssh --no-pager
sudo fail2ban-client status sshd
sudo ss -tulpn
```

Result:

- New kernel loaded successfully.
- Docker stack started automatically.
- SSH service was active.
- Fail2Ban SSH jail was active.
- No unexpected ports were exposed.

## Secret Rotation

Two runtime secrets were rotated after accidental exposure in terminal output:

- Cloudflare Tunnel token
- Guacamole PostgreSQL password

Rotation was completed without printing the new secret values.

Post-rotation validation:

- Cloudflared registered successfully with the tunnel.
- Guacamole continued working after PostgreSQL password rotation.

## Backup Exposure Status

Backup exposure review is intentionally deferred because backup infrastructure is not currently implemented.

When backups are later introduced, this must be revisited before backing up any secret-bearing paths.

## Final Hardening Status

Completed:

- SSH hardening
- Ansible access validation
- Ansible become validation
- Fail2Ban validation
- Listening port review
- Docker published-port review
- Docker network review
- Docker socket review
- Docker group review
- Compose file permission hardening
- Guacamole SSH key permission review
- Cloudflared update
- Container restart-policy validation
- Unattended upgrades review
- Kernel update and reboot validation
- Runtime secret rotation
- Final service validation

Deferred:

- Backup exposure review, because backup infrastructure is not currently in place

## Safe-to-Publish Notes

This public document should not include:

- Real IP addresses
- Real domains
- Real usernames if sensitive
- Tokens, passwords, or private keys
- Full rendered Compose files
- Vault contents
- Exact internal firewall source/destination IPs
