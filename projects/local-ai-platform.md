# Local AI platform: secure operations and observability

[Back to HomeLab](../README.md)

**Status:** Core platform deployed and documented. Subsequent routing/network changes remain separate work. Off-host backup and tested disaster recovery are deferred.

## Problem

A GPU workload depends on a tightly coupled kernel, NVIDIA driver, container runtime and model stack. Uncontrolled updates can break the platform, while broadly exposed model backends create unnecessary access paths.

## Implementation

I deployed a GPU-backed Linux VM on Proxmox with local model storage, Ollama, Open WebUI, Hermes and ComfyUI. Terraform manages virtual infrastructure and Ansible manages guest/application configuration. The platform uses restricted backend access, private HTTPS for configured browser services, an internal CA and source-scoped firewall rules.

The implementation also includes guarded storage initialization, protected package versions and a controlled maintenance workflow. Before critical-stack changes, rollback bundles record installed packages, manifests and checksums. These provide package-level recovery, not a replacement for VM/data backups.

## Validation and monitoring

- GPU operation checked in the guest and in GPU-enabled containers.
- Runtime service and container health checks after deployment.
- Prometheus collection of host, container and GPU metrics.
- A provisioned Grafana dashboard and Alertmanager notifications, including firing and resolved delivery tests.
- Ansible syntax/check-mode checks and documented clean reruns.

## Privacy scope

Local inference keeps model execution on the homelab. ComfyUI was separately configured for local/offline operation after downloads. This is not a claim that every platform component is air-gapped: metasearch, updates, downloads and alert delivery can involve external services. The later ComfyUI HTTPS route is not claimed complete merely because its certificate was prepared.

## Evidence

[Public AI Core technical write-up](https://github.com/t22n84r/homelab_docs/blob/main/ai_core/index.md). This case study summarizes implemented controls and retains the distinction between the completed platform milestone and later infrastructure work.
