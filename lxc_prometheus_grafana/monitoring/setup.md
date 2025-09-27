```markdown
# Monitoring – Setup

## LXC (Debian)
- Installed: `prometheus`, `prometheus-alertmanager`, `grafana`
- Prometheus targets:
  - `pve-node`: `<PVE_IP>:9100`
  - `pve`: `127.0.0.1:9221` (pve_exporter in venv)
- Config paths:
  - `/etc/prometheus/prometheus.yml`
  - `/etc/prometheus/alerts.yml`
  - PVE exporter config: `/etc/prometheus/pve.yml`
- Services: `systemctl enable --now prometheus alertmanager grafana-server pve-exporter`

## Proxmox host (exporters)
- `prometheus-node-exporter`, `lm-sensors`, `smartmontools`, `nvme-cli`
- Kernel modules persisted: `k10temp`, `asus_wmi_sensors`
- SMART textfile:
  - Script: `/usr/local/sbin/smart_textfile.sh`
  - Output: `/var/lib/node_exporter/textfile_collector/smart.prom` (mode 644)
  - Cron: `/etc/cron.d/smart_textfile` → every 5 min
- Verified:
  - `curl http://<PVE_IP>:9100/metrics` has `node_hwmon_*`
  - `smart.prom` contains `smart_device_temperature_celsius`, `smart_device_healthy`

## Grafana
- URL: `http://<LXC_IP>:3000`
- Datasource: Prometheus → `http://localhost:9090`
- Imported dashboards: Node Exporter Full
