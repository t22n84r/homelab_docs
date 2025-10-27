DNSSEC validation, aggressive caching, prefetching

Robust hardening (qname minimisation, NXDOMAIN harden, etc.)

Generous caches/threads (uses ~400–600 MB RAM comfortably on a 2 GiB Pi 4)

Optional remote control & stats

Optional LAN access (for testing) while still using Pi-hole → Unbound in production

1) Install Unbound
sudo apt update
sudo apt install -y unbound

2) Root hints (and keep them fresh)
sudo mkdir -p /var/lib/unbound
sudo wget -O /var/lib/unbound/root.hints https://www.internic.net/domain/named.root


(Optional) auto-refresh monthly:

echo 'wget -qO /var/lib/unbound/root.hints https://www.internic.net/domain/named.root && systemctl reload unbound' | sudo tee /etc/cron.monthly/unbound-root-hints > /dev/null
sudo chmod +x /etc/cron.monthly/unbound-root-hints

3) Create the Unbound config
    leave the original pi-hole.conf as is
    add a new file for new settings sudo nano /etc/unbound/unbound.conf.d/10-tuning.conf

4) Enable + start Unbound
sudo systemctl enable --now unbound
sudo systemctl restart unbound

5) Point Pi-hole to Unbound

Pi-hole Admin → Settings → DNS:

Upstream DNS Servers → Custom: 127.0.0.1#5335

DNSSEC in Pi-hole: OFF (Unbound already validates)

Leave conditional forwarding off unless you specifically need local names from your router.

6) Optional: Unbound remote control & stats

This lets you query live stats without restarting.

Edit (new file) to enable control:

sudo nano /etc/unbound/unbound.conf.d/remote-control.conf

remote-control:
  control-enable: yes


Generate keys & restart:

sudo unbound-control-setup
sudo systemctl restart unbound


Query:

sudo unbound-checkconf
sudo systemctl restart unbound
sudo systemctl status --no-pager unbound
sudo unbound-control status
sudo unbound-control stats_noreset | egrep 'total.num.queries|cache|.*recursion.*avg'
pihole restartdns = do it from web admin
sudo systemctl reload unbound
Test from a LAN client (not the Pi):

Windows: nslookup cloudflare.com <PIHOLE_IP>

Linux/macOS: dig @<PIHOLE_IP> cloudflare.com

7) Sanity tests
# Ask Unbound directly (localhost:5335)
dig +short @127.0.0.1 -p 5335 cloudflare.com

# DNSSEC working? (dnssec-failed.org should SERVFAIL)
dig +dnssec @127.0.0.1 -p 5335 dnssec-failed.org

# AD (Authenticated Data) flag present on valid answers?
dig +dnssec @127.0.0.1 -p 5335 cloudflare.com | grep 'flags'  # look for 'ad'

# Through Pi-hole (port 53) after wiring:
dig +short cloudflare.com

Why this setup = “most freedom”

No forwarding: you resolve from the root—maximum independence and privacy.

DNSSEC: end-to-end validation.

Hardening on: reduces data leakage and spoofing issues.

Generous threads/caches: uses your spare RAM/CPU to deliver near-instant answers, even under load.

LAN testing: you can query Unbound directly for diagnostics without disrupting Pi-hole’s role on port 53.
