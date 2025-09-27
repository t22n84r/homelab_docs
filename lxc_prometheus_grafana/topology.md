  PVE[Proxmox Host<br/>(B550 + 5900X)] <br/>
  LXC[Monitoring LXC (Debian)] <br/>
  Prom[Prometheus :9090] <br/>
  AM[Alertmanager :9093] <br/>
  Gf[Grafana :3000] <br/>
  nodeExp[node_exporter :9100<br/>+ sensors] <br/>
  smart[SMART textfile collector] <br/>
  pveExp[PVE exporter :9221] <br/>

  PVE --> LXC
  LXC --> Prom
  LXC --> AM
  LXC --> Gf
  Prom --> nodeExp
  Prom --> smart
  Prom --> pveExp
