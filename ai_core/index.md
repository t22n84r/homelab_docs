# Local AI Core Platform: Private GPU Inference, Agent Workloads, and Observability

## Project summary

I designed and deployed a local-first AI platform on a Proxmox homelab. The platform combines GPU-accelerated model inference, a browser-based chat interface, an agent environment, image generation, private metasearch, guarded lifecycle management, and end-to-end monitoring.

The implementation uses infrastructure as code and configuration management rather than a collection of one-off shell commands. Proxmox virtual hardware is managed separately from guest and application configuration, while sensitive values remain encrypted.

## Goals

- Run modern AI inference locally on a dedicated NVIDIA GPU.
- Provide friendly browser access without exposing model backends directly.
- Support both conversational AI and terminal-style agent workflows.
- Keep model data and application state on dedicated local storage.
- Make kernel, GPU-driver, and container-runtime updates recoverable.
- Restrict services by source and purpose rather than opening broad LAN access.
- Add useful dashboards and actionable email alerts.

## Architecture

The core workload runs in an Ubuntu virtual machine with a passed-through RTX-class GPU and dedicated model SSD. A separate automation host manages the environment with Ansible, and a restricted Debian LXC runs Prometheus, Grafana, and Alertmanager.

The application layer includes:

- Ollama for local model serving
- Open WebUI for conventional browser chat
- Hermes Agent for terminal-style agent sessions
- SearXNG for private metasearch integration
- ComfyUI for GPU-accelerated image generation
- Caddy for private HTTPS and hostname-based routing

Application containers communicate on an internal Docker network. Backend services such as model inference and metasearch are not exposed as general LAN endpoints.

## GPU and container runtime

The NVIDIA stack uses distribution-provided prebuilt open kernel modules instead of DKMS. Secure Boot state, GPU identity, driver operation, and Nouveau blacklisting are all validated before deployment proceeds.

NVIDIA Container Toolkit provides CDI device discovery. Validation occurs both on the host and inside GPU-enabled containers so that a healthy driver cannot mask a broken container-runtime path.

## Storage safety

The model disk is managed by an Ansible workflow that validates the stable device path, expected serial, capacity range, existing filesystem type, label, and UUID before allowing destructive initialization. Destructive operations require explicit authorization flags.

Large model data is separated from the operating-system disk. Disk thresholds are monitored, and Docker image growth is reviewed with targeted cleanup rather than broad pruning.

## Secure application access

The browser applications use private DNS names and certificates issued by an internal certificate authority. One reverse proxy routes multiple HTTPS names to their respective services.

Firewall policy limits:

- SSH to administrative and automation sources
- Web applications to approved administrative clients
- Monitoring exporters to the monitoring host
- SMTP submission to the alerting service only

Administrative users are intentionally excluded from the Docker group because Docker socket access is equivalent to root.

## Controlled maintenance and recovery

The critical platform components—kernel, NVIDIA driver, Docker, container runtime, and GPU container toolkit—are protected from ordinary unattended upgrades with package holds.

A custom maintenance interface provides:

- Health and capacity status
- Safe operating-system updates that preserve critical holds
- Controlled critical-stack updates
- Explicit rollback of the latest critical-stack change
- Dry-run and repair of maintenance guardrails

Before a critical update, the system creates a timestamped rollback bundle containing repacked installed packages, version manifests, package selections, hold state, and checksums. A retention policy keeps the newest complete bundles and protects the latest known-good marker.

These bundles provide package-level recovery. They are deliberately not described as full backups; off-host disaster recovery is a separate infrastructure requirement.

## Monitoring and alerting

Prometheus collects:

- Host CPU, memory, disk, and network metrics
- Container CPU and memory metrics
- NVIDIA GPU utilization, temperature, and error metrics

Grafana receives a provisioned AI Core dashboard. Alertmanager sends authenticated TLS email notifications through an external mail provider.

Alerts cover exporter failure, missing application containers, root-disk pressure, sustained CPU or memory usage, GPU temperature, and NVIDIA XID errors. Both firing and resolved notification delivery were tested.

The monitoring container intentionally remains non-nested and AppArmor-confined. Where package-provided systemd sandboxing conflicted with LXC namespace restrictions, targeted service overrides disabled only the incompatible inner namespace controls instead of weakening the entire container.

## Automation quality

The final implementation includes:

- Syntax checks and check-mode previews before changes
- Assertions for platform, device, package, and security assumptions
- Explicit check-mode boundaries for runtime-only work
- Runtime health checks after deployment
- Idempotency verification through clean reruns
- Encrypted secret variables and protected rendered configuration
- Modular playbooks for storage, GPU, Docker, applications, maintenance, PKI, monitoring, dashboards, and alerts

## Outcome

The completed platform provides private GPU inference, browser chat, agent sessions, image generation, metasearch, private HTTPS, controlled upgrades, local package rollback, live dashboards, and email alerting. The remaining infrastructure gap is full off-host backup and tested disaster recovery, which is explicitly tracked rather than obscured by the package rollback mechanism.

