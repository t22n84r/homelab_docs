  PVE[Proxmox Host(B550 + 5900X)] <br/>
  
  LXC[Monitoring LXC (Debian)] <br/>
  
  Prom[Prometheus :9090] <br/>
  
  AM[Alertmanager :9093] <br/>
  
  Gf[Grafana :3000] <br/>
  
  nodeExp[node_exporter :9100 + sensors] <br/>
  
  smart[SMART textfile collector] <br/>
  
  pveExp[PVE exporter :9221] <br/>

  PVE --> LXC <br/>
  LXC --> Prom, AM, Gf <br/>
  Prom --> nodeExp, smart, pveExp
