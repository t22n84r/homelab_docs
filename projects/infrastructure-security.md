# Infrastructure security through a working homelab

[Back to HomeLab](../README.md)

**Scope:** Independently operated Proxmox environment. Core services and hardening are implemented. The core Management/Servers cutover and Ansible/Terraform reconciliation are complete; independent off-host recovery remains deferred.

## Design

The environment separates application workloads, a dedicated automation host, monitoring and a remote-access gateway. The security work focuses on identity, reachability, secret handling and validation after changes.

| Boundary | Implemented work | Validation or limitation |
|---|---|---|
| Administrative access | Key-based SSH, disabled root/password SSH where documented, source-scoped firewall access and administrator MFA | Access and service checks recorded in hardening notes |
| Automation identity | Separate human administrator, local runner, Ansible SSH account and Terraform API identity | Ansible connectivity/become and Terraform API planning validated |
| Secrets | Ansible Vault, restricted runtime files and secret material outside public source | Backup exposure review and tighter host-side sudo scoping remain open |
| Remote access | Historical Guacamole behind Cloudflare Tunnel/Access; internal container networking and no broad service port publication | Original validation documented; gateway currently stopped, with autostart disabled and guest address unverified |
| Service isolation | Proxmox firewall policy and purpose-specific service reachability | Audit verified managed-host connectivity and Terraform alignment; older firewall snapshots retain historical labels |
| Network modernization | UniFi gateway deployment, isolated lab network and Management/Servers cutover | Core cutover complete; migrated IaC validated and desktop/automation copies synchronized |

## Engineering outcomes

The automation workflow separates platform changes from guest configuration and keeps sensitive values out of public examples. Public records document the Proxmox Terraform provider migration, dedicated runner model and successful access validation. The completed core cutover was followed by a legacy-address audit of the IaC repository. All three inventory hosts answered Ansible ping, affected playbooks passed syntax checks, and a full Terraform plan reported no changes. The [network audit record](network-migration-audit.md) describes the evidence and limits.

Remote-access hardening includes reviewing listeners and container networks, restricting secret files, maintaining key-based administration and checking recovery after reboot. The public source records runtime-secret rotation after accidental terminal disclosure, demonstrating operational follow-through rather than treating file permissions as sufficient on their own.

## Public implementation records

- [Automation control-node hardening](https://github.com/t22n84r/homelab_docs/blob/main/automation_vm/automation-hardening.md)
- [Terraform provider migration](https://github.com/t22n84r/homelab_docs/blob/main/automation_vm/terraform-provider-swap-public.md)
- [Remote-access gateway hardening](https://github.com/t22n84r/homelab_docs/blob/main/gateway_vm/remote-access-gateway-hardening-public.md)

These records describe particular milestones. Historical examples are not automatically current deployment guidance. They do not establish a deployed AWX workflow or tested off-host disaster recovery. The dormant gateway's provisional target remains separate from verified guest addressing.
