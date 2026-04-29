# Infrastructure Automation Control Node Hardening (Public)

## Scope
This document summarizes the hardened design for a dedicated infrastructure automation control node and its Proxmox-side automation access path.

## Architecture Summary
- A dedicated **automation VM** is used as the control node.
- A human admin account is used for administration and maintenance.
- A separate **local runner account** is used for Terraform and Ansible execution.
- Proxmox platform automation is split from host OS automation:
  - **Terraform** uses a dedicated Proxmox API token.
  - **Ansible** uses a dedicated SSH automation user on the Proxmox host.

## Identity Model
### Automation VM
- **Human admin account**: interactive administration, editing, troubleshooting, maintenance.
- **Runner account**: executes Terraform and Ansible, reads only the required secret material, and does not retain local sudo access.

### Proxmox Host
- **Human admin account**: normal host administration.
- **SSH automation account**: used by Ansible for host-maintenance automation.
- **API automation account/token**: used by Terraform for Proxmox API operations.

## Filesystem Layout
A dedicated shared automation workspace is used instead of placing long-term automation assets under a personal home directory.

### High-level layout
- Shared automation workspace
- Separate secrets directory
- Ansible project tree
- Terraform project tree

## SSH Hardening
### Automation VM
- Root SSH login disabled.
- Password SSH disabled.
- Public key authentication enabled.
- Only the designated human admin account is allowed over SSH.
- Root is reserved for break-glass/local recovery only.

### Proxmox Host
- Root SSH login disabled.
- Password SSH disabled.
- Public key authentication enabled.
- SSH automation account is explicitly allowed.
- Automation SSH key is restricted to the automation VM source IP.

## Root Account Policy
### Automation VM root
- Strong local password set.
- Not used for normal remote administration.
- Reserved for local console / break-glass recovery.

### Proxmox host root
- Not used for normal SSH administration.
- Reserved for break-glass / console / recovery scenarios.

## Secret Handling Model
### Terraform
- API token material is stored outside the Terraform project directory.
- Terraform executes through the dedicated runner account.
- Terraform runtime artifacts are owned by the runner account.

### Ansible
- Non-secret variables are separated from secret variables.
- Secret variables are encrypted with **Ansible Vault**.
- The Vault password file is stored outside the repo and readable only by the runner account.
- Ansible `become` works through Vault-backed secret retrieval.

## Access Boundary for Secrets
- Shared project files are readable by the expected admin/automation model.
- Secret files are not directly readable by the normal human account without sudo.
- Secret files are readable by the runner account only where required for execution.
- A sudo-capable admin can still read secrets by becoming root; this is an accepted property of the local trust model.

## Firewall Model
### Design choice
The primary network boundary is enforced at the **Proxmox firewall layer**, not via guest firewalld inside the automation VM.

### Proxmox host
Dedicated rules allow the automation VM to reach only the required services on the Proxmox host:
- SSH for Ansible
- Proxmox API for Terraform

### Automation VM
Inbound SSH is restricted at the Proxmox firewall layer to trusted admin desktop IPs.

## Validation Summary
The hardened setup was revalidated successfully:
- Ansible SSH path works.
- Ansible `become` works through Vault.
- Terraform validates successfully.
- Terraform plan successfully reaches the Proxmox API.
- Secret files remain usable by the runner account after permission tightening.
- The local runner account on the automation VM no longer has sudo.

## Deferred Items
These were intentionally left as future work:
- Scope down Proxmox-side automation sudo privileges based on observed long-term playbook needs.
- Review backup exposure of the secrets directory before any backup system is introduced.
- Review Terraform configuration drift separately before apply.

## Suggested Files to Put in GitHub
Good candidates:
- Public hardening summary document
- `ansible.cfg`
- Inventory examples with sensitive values removed or templated
- Non-secret `group_vars` files
- Terraform source files (`main.tf`, `variables.tf`, modules)
- Terraform `.gitignore`
- Example playbooks/roles without secrets
- Sanitized firewall/rule examples
- Validation checklist / runbook

## Files to Avoid Putting in GitHub
Do **not** publish:
- Vault password file
- Vault-encrypted secrets if the repo is public and the companion unlock path exists elsewhere
- API token env files
- Private keys
- `authorized_keys`
- Terraform state files
- `.terraform/` provider directory
- Anything containing internal IPs if you want the repo fully public
