# Topology

```mermaid
flowchart LR
  PVE[Proxmox Host (B550 + 5900X)]
  LXC[Monitoring LXC (Debian)]
  Prom[Prometheus :9090]
  AM[Alertmanager :9093]
  Gf[Grafana :3000]
  nodeExp[node_exporter :9100 + sensors]
  smart[SMART textfile collector]

  PVE --> LXC
  LXC --> Prom
  LXC --> AM
  LXC --> Gf
  Prom --> nodeExp
  Prom --> smart
  Prom --> pveExp[PVE exporter :9221]
