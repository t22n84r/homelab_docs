# Infrastructure security through a working homelab

[Back to HomeLab](../README.md)

**Scope:** Independently operated Proxmox environment. Core services and hardening are implemented; network migration and automation reconciliation remain in progress.

## Design

The environment separates application workloads, a dedicated automation host, monitoring and a remote-access gateway. The security work focuses on identity, reachability, secret handling and validation after changes.

| Boundary | Implemented work | Validation or limitation |
|---|---|---|
| Administrative access | Key-based SSH, disabled root/password SSH where documented, source-scoped firewall access and administrator MFA | Access and service checks recorded in hardening notes |
| Automation identity | Separate human administrator, local runner, Ansible SSH account and Terraform API identity | Ansible connectivity/become and Terraform API planning validated |
| Secrets | Ansible Vault, restricted runtime files and secret material outside public source | Backup exposure review and tighter host-side sudo scoping remain open |
| Remote access | Guacamole behind Cloudflare Tunnel/Access; internal container networking and no broad service port publication | SSH, container, tunnel and reboot validation documented |
| Service isolation | Proxmox firewall policy and purpose-specific service reachability | Rule changes must be rechecked during network migration |
| Network modernization | UniFi gateway deployment, isolated lab network and staged workload migration | Full management/server migration is not complete |

## Engineering outcomes

The automation workflow separates platform changes from guest configuration and keeps sensitive values out of public examples. Public records document the Proxmox Terraform provider migration, dedicated runner model and successful access validation. Recent migration work includes moving the AI workload to its server network and checking monitoring health afterwards; that does not imply that all infrastructure has migrated.

Remote-access hardening includes reviewing listeners and container networks, restricting secret files, maintaining key-based administration and checking recovery after reboot. The public source records runtime-secret rotation after accidental terminal disclosure, demonstrating operational follow-through rather than treating file permissions as sufficient on their own.

## Public implementation records

- [Automation control-node hardening](https://github.com/t22n84r/homelab_docs/blob/main/automation_vm/automation-hardening.md)
- [Terraform provider migration](https://github.com/t22n84r/homelab_docs/blob/main/automation_vm/terraform-provider-swap-public.md)
- [Remote-access gateway hardening](https://github.com/t22n84r/homelab_docs/blob/main/gateway_vm/remote-access-gateway-hardening-public.md)

These records describe particular milestones. Historical examples are not automatically current deployment guidance. They do not establish a deployed AWX workflow, a fully migrated network or tested off-host disaster recovery.
