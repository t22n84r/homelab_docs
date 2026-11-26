# Network migration and configuration reconciliation

[Back to HomeLab](../README.md)

**Reconciled:** November 25, 2026.

The core infrastructure cutover to separate Management and Servers networks is complete. The Ansible/Terraform repository was audited for legacy addresses and subnet references, and both the desktop and automation working copies were synchronized to the merged correction.

## Configuration changes and evidence

The audit corrected the hypervisor inventory and Terraform endpoint, the AI maintenance SSH target, and monitoring references. Monitoring variables and Prometheus templates now derive relevant addresses from the inventory to reduce future duplication.

All three managed inventory hosts answered Ansible ping after host keys were verified against keys read from the systems. The affected monitoring playbooks passed syntax checks. Terraform validation succeeded, the effective hypervisor endpoint was checked, and a full plan reported **No changes**.

These results establish the checked configuration and connectivity baseline. They do not establish fresh validation of every application endpoint, DNS record, certificate SAN, NAS mount, or firewall rule.

## Dormant gateway

The earlier Cloudflare/Guacamole gateway remains powered off with autostart disabled. Its VLAN setting matches the hypervisor. A client-side SSH target was updated to a user-selected provisional address; the guest's static address and DHCP reservation remain unverified. Historical gateway walkthroughs describe its original working setup rather than a currently available remote-access service.

## Documentation treatment

Active private runbooks use the migrated service addresses. Original command output, hardening snapshots, certificate examples, and retired DNS configuration retain historical labels where appropriate. This public repository uses placeholders for environment-specific addresses.

Pi-hole/Unbound is retired. Historical DNS setup guides do not represent the current resolver architecture. Independent off-host backup and tested full-system restore remain deferred.
