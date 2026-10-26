# Taanvir Nazmee · Infrastructure & Cybersecurity Portfolio

Practical work in Linux infrastructure, security controls, automation and operational troubleshooting. This repository brings together my independently operated homelab and authorized cybersecurity training.

## Start with the portfolio

**[Cybersecurity & Infrastructure Security Portfolio](https://github.com/t22n84r/CyberSec)** — a curated overview with 17 web-security exercises, three practical lab reports, infrastructure case studies and an evidence register.

- **15 web labs recorded completed:** access control, authentication, SQL injection and SSRF.
- **Two unconfirmed investigations:** blind SSRF exercises, explicitly separated from completed results.
- **Practical evidence:** secure file transfer and TLS analysis, Metasploitable enumeration/root-shell validation, and Windows endpoint-security observations.

## Selected homelab work

| Project | Engineering focus | Record |
|---|---|---|
| Local AI Core | Restricted GPU platform access, controlled maintenance, package rollback and observability | [Technical overview](ai_core/index.md) |
| Automation control node | Separate identities, Ansible Vault, SSH boundaries and Terraform access | [Hardening](automation_vm/automation-hardening.md) |
| Terraform provider migration | Proxmox resource conversion and validation | [Migration record](automation_vm/terraform-provider-swap-public.md) |
| Remote-access gateway | Guacamole, Cloudflare Tunnel/Access, container exposure and secret-file handling | [Hardening and validation](gateway_vm/remote-access-gateway-hardening-public.md) |
| Monitoring recovery | LXC/AppArmor and systemd compatibility, unprivileged services and HTTPS trust | [Recovery runbook](lxc_prometheus_grafana/monitoring/monitoring-lxc-hardening-public.md) |
| Media service | Container exposure, NAS access and runtime hardening | [Jellyfin hardening](jellyfin/jellyfin-lxc-hardening-public.md) |

## Current scope

The AI platform's core deployment and monitoring milestone is complete. The broader management/server network migration, automation drift reconciliation and independent off-host recovery work remain active or deferred. Package rollback is not a full backup strategy.

Older DNS records describe historical Pi-hole/Unbound work; those services have since been retired. Implementation records describe their original milestones and may include environment-specific assumptions. The curated portfolio distinguishes implemented controls, historical work and proposed next steps.

## Evidence and privacy

The new portfolio uses sanitized text, clearly labeled observations and explicit completion limits. Original screenshots, credentials, TLS session keys and private operational configurations are excluded from the new portfolio. Existing implementation records are linked as supporting work; their publication is not a fresh audit of every historical file.

[Evidence register](https://github.com/t22n84r/CyberSec/blob/main/evidence/README.md) · [Publication policy](https://github.com/t22n84r/CyberSec/blob/main/docs/publication-policy.md) · [LinkedIn](https://www.linkedin.com/in/taanvir-nazmee-5b13a7106/)

Reviewed October 26, 2026. Independent projects and training are distinct from employer work.
