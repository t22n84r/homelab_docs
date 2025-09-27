```bash
systemctl status prometheus alertmanager grafana-server pve-exporter

curl -s http://127.0.0.1:9090/api/v1/targets | jq '.data.activeTargets[].labels'
