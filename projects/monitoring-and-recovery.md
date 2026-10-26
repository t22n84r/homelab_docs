# Recovering monitoring without removing the container boundary

[Back to HomeLab](../README.md)

**Status:** Service recovery and internal HTTPS validation documented as completed.

## Failure

After a host kernel upgrade and reboot, monitoring services inside a Proxmox LXC failed with `226/NAMESPACE`. Application configuration and binaries were intact. Host logs showed AppArmor denying namespace and mount operations attempted by inner systemd sandbox settings.

## Decision and fix

I used service-specific systemd drop-ins to remove the incompatible inner namespace controls while retaining LXC confinement, disabled nesting and dedicated unprivileged service identities. Compatible process restrictions remained. A runtime directory resolved an exporter working-directory issue, and redundant container-incompatible units were retired.

Grafana’s health checks were also corrected to use HTTPS and the configured hostname. Installing the public internal-CA certificate enabled certificate verification without a permanent insecure bypass.

## Recorded outcome

Prometheus, Grafana and the PVE exporter returned to service; Prometheus targets were healthy, HTTPS validation worked and no failed systemd units remained. The public runbook includes rollback guidance and checks to repeat after future kernel, LXC or service upgrades.

## Why this matters

Hardening directives interact with the environment that hosts them. A secure operational decision needs a root-cause explanation, a scoped exception and evidence that the remaining boundary still works. A configuration checklist alone would not have resolved this failure.

[Full implementation and validation record](https://github.com/t22n84r/homelab_docs/blob/main/lxc_prometheus_grafana/monitoring/monitoring-lxc-hardening-public.md).
