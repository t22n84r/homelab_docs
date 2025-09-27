# Cheatsheet

## Proxmox host
```bash
# sensors & SMART
sensors
/usr/local/sbin/smart_textfile.sh
tail /var/lib/node_exporter/textfile_collector/smart.prom

# services
systemctl status prometheus-node-exporter
journalctl -u prometheus-node-exporter -n 200 --no-pager
