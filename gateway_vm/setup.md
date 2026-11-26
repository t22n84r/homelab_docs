# Remote Access Gateway VM Setup — Public Notes

> **Status reconciled November 25, 2026 — historical gateway record.** The gateway VM is currently stopped with autostart disabled. Its VLAN is reconciled, while the provisional client-side address has not been verified in the guest or DHCP reservation. The working-service, tunnel, access-rule, and reboot results below belong to the original milestone. See the [network audit record](../projects/network-migration-audit.md).

## Purpose

Build a dedicated remote-access gateway VM to provide browser-based access to internal systems without exposing home network services directly to the internet.

## High-Level Architecture

```text
Browser
  -> Cloudflare Access
  -> Cloudflare Tunnel
  -> Remote Access Gateway VM
  -> Guacamole / SSH / VNC / internal services
```

## Design Decisions

- Use a dedicated gateway VM instead of installing remote-access tooling directly on the hypervisor.
- Use Cloudflare Tunnel for outbound-only connectivity instead of router port forwarding.
- Use Cloudflare Access as the outer identity gate before Guacamole.
- Use Guacamole as the browser-accessible internal access hub.
- Avoid client VPN dependency for work-device compatibility.
- Use Proxmox firewalling for the gateway VM rather than UFW inside the gateway at this stage.
- Keep gateway SSH hardening as a separate follow-up phase after access paths are stable.

## VM Provisioning Summary

- VM was created on Proxmox using Terraform.
- Terraform provider used: BPG Proxmox provider.
- Ubuntu Server was installed manually from ISO due to cloud-image availability issues during setup.
- VM was configured as a normal server VM, not a container.
- ISO/CD-ROM boot was used for installation, then normal disk boot was used after install.

## Ansible Setup for This VM

Project layout:

```text
/srv/automation/projects/ansible/remote-access-gateway/
├── ansible.cfg
├── inventory.ini
├── group_vars/
│   └── remote_access_gateway/
│       ├── main.yml
│       └── vault.yml
├── playbooks/
└── templates/
```

The automation workflow runs Ansible from the automation VM under the dedicated automation user.

Standard command pattern:

```bash
sudo -u automation -H bash -lc '
source /home/automation/venvs/ansible/bin/activate &&
cd /srv/automation/projects/ansible/remote-access-gateway &&
ansible-playbook playbooks/<playbook>.yml --vault-password-file /srv/automation/secrets/ansible-vault.pass
'
```

## Ansible Vault / Secrets Model

- Vault password file unlocks encrypted vault files.
- Actual secrets are stored inside encrypted `vault.yml`, not inside the vault password file.
- Readable group vars reference vault variables without exposing secret values.

Readable vars pattern:

```yaml
---
ansible_become_method: sudo
ansible_become_password: "{{ vault_remote_gateway_ansible_become_password }}"

remote_gateway_stack_dir: "/opt/remote-access-gateway"

guacamole_version: "1.6.0"
postgres_version: "16"

guacamole_postgres_db: "guacamole_db"
guacamole_postgres_user: "guacamole_user"
guacamole_postgres_password: "{{ vault_guacamole_postgres_password }}"

cloudflare_tunnel_token: "{{ vault_cloudflare_tunnel_token }}"
```

## Baseline Configuration Applied

Baseline playbook installed and enabled core system tooling.

Installed packages included:

- `qemu-guest-agent`
- `fail2ban`
- `unattended-upgrades`
- `micro`
- `btop`
- `git`
- `curl`
- `ca-certificates`
- `gnupg`
- `lsb-release`
- `dnsutils`
- `net-tools`
- `jq`

Services enabled:

- `qemu-guest-agent`
- `fail2ban`
- `unattended-upgrades`

Intentionally excluded:

- `ufw` — Proxmox firewall will handle VM firewalling.
- `vim` — preference is `micro`.
- `htop` — preference is `btop`.

## Validation Completed

- SSH from automation VM to gateway `ansible` user succeeded.
- Ansible ping succeeded.
- Ansible become/sudo succeeded using vault-backed become password.
- Baseline playbook completed successfully.
- Required services were active after baseline.

## Safe to Commit Publicly

Usually safe:

- Sanitized Terraform examples.
- Sanitized Ansible playbooks.
- Sanitized `ansible.cfg`.
- Sanitized inventory examples with placeholder IPs.
- Non-secret documentation.

Do not commit:

- Vault password file.
- Encrypted vault files if repository is public.
- Private keys.
- Tokens.
- Real IP/domain details if treating home topology as private.
