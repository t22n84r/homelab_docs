# Taanvir Nazmee · HomeLab Portfolio

My independently operated homelab brings together Linux infrastructure, virtualization, automation, networking and local AI services. This repository documents how I build, harden, monitor and recover those systems.

## Start with the project summaries

| Project | Engineering focus | Overview |
|---|---|---|
| Infrastructure and automation | Proxmox, separate automation identities, access boundaries, secrets and segmented networks | [Infrastructure summary](projects/infrastructure-security.md) |
| Local AI platform | GPU operations, restricted service access, controlled maintenance and observability | [AI platform summary](projects/local-ai-platform.md) |
| Monitoring recovery | Diagnosing LXC/AppArmor and systemd conflicts, restoring services and validating HTTPS | [Monitoring summary](projects/monitoring-and-recovery.md) |

## Implementation records

| Project | Record |
|---|---|
| AI Core | [Technical overview](ai_core/index.md) |
| Automation control node | [Hardening](automation_vm/automation-hardening.md) |
| Terraform provider migration | [Migration record](automation_vm/terraform-provider-swap-public.md) |
| Remote-access gateway | [Hardening and validation](gateway_vm/remote-access-gateway-hardening-public.md) |
| Monitoring recovery | [Recovery runbook](lxc_prometheus_grafana/monitoring/monitoring-lxc-hardening-public.md) |
| Media service | [Jellyfin hardening](jellyfin/jellyfin-lxc-hardening-public.md) |

## Current scope

The AI platform's core deployment and monitoring milestone is complete. The core Management/Servers network cutover and Ansible/Terraform reconciliation are complete, with both IaC working copies synchronized. See the [network audit record](projects/network-migration-audit.md) for the validation evidence and dormant gateway exception. Independent off-host backup and tested full-system recovery remain deferred. Package rollback is not a full backup strategy.

Older DNS records describe historical Pi-hole/Unbound work; those services have since been retired. Implementation records describe their original milestones and may include environment-specific assumptions.

The project summaries use sanitized descriptions and distinguish implemented controls from proposed next steps. Existing implementation records remain supporting documentation; the network audit record identifies the checks performed and their limits.

[LinkedIn](https://www.linkedin.com/in/taanvir-nazmee-5b13a7106/)

Network status reconciled November 25, 2026. These are independent homelab projects, separate from employer work.
