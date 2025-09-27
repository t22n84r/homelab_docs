  PVE[Proxmox Host<br/>(B550 + 5900X)];
  LXC[Monitoring LXC (Debian)]
  Prom[Prometheus :9090]
  AM[Alertmanager :9093]
  Gf[Grafana :3000]
  nodeExp[node_exporter :9100<br/>+ sensors]
  smart[SMART textfile collector]
  pveExp[PVE exporter :9221]

  PVE --> LXC
  LXC --> Prom
  LXC --> AM
  LXC --> Gf
  Prom --> nodeExp
  Prom --> smart
  Prom --> pveExp
