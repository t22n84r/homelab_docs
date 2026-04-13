## 1. What we did to make it work

### A. Installed working mitmproxy standalone binary on Proxmox

```bash
mkdir -p ~/apps/mitmproxy
cd ~/apps/mitmproxy
curl -LO https://current_version_mitmproxy
tar -xzf current_version_mitmproxy
```

### B. Started mitmweb on the Proxmox host for the BlissOS VM

Use the **Proxmox host IP**, not the VM IP.

```bash
cd ~/apps/mitmproxy
./mitmweb --listen-host <PROXMOX_HOST_IP> --listen-port 8080
```

### C. Verified mitmproxy was listening

```bash
ss -ltnp | grep 8080
```

### D. Set the global Android proxy in BlissOS over ADB

```bash
adb shell settings put global http_proxy <PROXMOX_HOST_IP>:8080
adb shell settings get global http_proxy
```

### E. Confirmed the VM was reaching mitmproxy

Observed mitmproxy logs showing client connections from the BlissOS VM IP.

### F. Installed mitmproxy CA as a user cert in BlissOS

Used BlissOS **Install certificate** in Settings.

### G. Created Android system-CA hashed certificate filename on Proxmox

```bash
cd ~/.mitmproxy
openssl x509 -inform PEM -subject_hash_old -in mitmproxy-ca-cert.cer | head -1
cp mitmproxy-ca-cert.cer <HASH>.0
ls -l <HASH>.0
```

### H. Pushed the hashed cert file into BlissOS

You used the actual filename directly.

```bash
adb push ~/.mitmproxy/<HASH>.0 /sdcard/Download/
```

### I. Verified BlissOS root filesystem and CA path

```bash
adb shell
mount | grep -E ' /system | / | cacerts'
ls -ld /system/etc/security/cacerts
ls -ld /apex/com.android.conscrypt/cacerts 2>/dev/null
```

### J. Installed mitmproxy CA into the BlissOS system CA store

```bash
adb shell
cp /sdcard/Download/<HASH>.0 /system/etc/security/cacerts/
chmod 644 /system/etc/security/cacerts/<HASH>.0
chown root:root /system/etc/security/cacerts/<HASH>.0
ls -l /system/etc/security/cacerts/<HASH>.0
reboot
```

### K. Re-verified after reboot

```bash
adb shell settings get global http_proxy
adb shell ls -l /system/etc/security/cacerts/<HASH>.0
ss -ltnp | grep 8080
```

### L. Enabled SSH port-forwarding for mitmproxy Web UI on Proxmox

You changed SSH config to allow only the specific Web UI target.
AllowTcpForwarding local
PermitOpen 127.0.0.1:8081

### M. Opened SSH tunnel from your desktop to mitmproxy Web UI

```bash
ssh -L 8081:127.0.0.1:8081 user@<PROXMOX_HOST_IP>
```

### N. Used mitmproxy Web UI to capture the link

---

## 2. Full undo / fully remove everything

### A. Stop mitmproxy / mitmweb

If still running in foreground, close the terminal.
Or:

```bash
pkill -f mitmweb
pkill -f mitmproxy
```

### B. Verify ports are no longer listening

```bash
ss -ltnp | grep -E '8080|8081'
```

### C. Clear the global proxy in BlissOS

```bash
adb shell settings delete global http_proxy
adb shell settings get global http_proxy
```

### D. Remove the mitmproxy system CA from BlissOS

```bash
adb shell
rm /system/etc/security/cacerts/<HASH>.0
reboot
```

### E. Optionally remove the user-installed mitmproxy cert in BlissOS

Remove it from BlissOS certificate settings manually.

### F. Revert SSH forwarding allowance on Proxmox

Edit SSH config and remove or revert the forwarding change you added for mitmproxy Web UI.
Then:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

### G. Optionally remove the standalone mitmproxy files

```bash
rm -rf ~/apps/mitmproxy
```

### H. Optionally remove mitmproxy runtime cert material on Proxmox

```bash
rm -rf ~/.mitmproxy
```

---

## 3. Keep for repeated use

### Leave ON / keep in place

#### A. Keep the mitmproxy system CA installed in BlissOS

Do **not** remove:

```text
/system/etc/security/cacerts/<HASH>.0
```

#### B. Keep the SSH forwarding allowance for mitmproxy Web UI on Proxmox

Do **not** revert the specific SSH config change you made for the Web UI target.

#### C. Keep the standalone mitmproxy files on Proxmox

Do **not** remove:

```text
~/apps/mitmproxy
```

### Turn OFF after each use

#### A. Stop mitmproxy / mitmweb

Close the terminal running it, or:

```bash
pkill -f mitmweb
pkill -f mitmproxy
```

#### B. Confirm the proxy port is no longer listening

```bash
ss -ltnp | grep 8080
```

#### C. Clear the global Android proxy in BlissOS after each session

```bash
adb shell settings delete global http_proxy
adb shell settings get global http_proxy
```

### Re-enable for the next use

#### A. Start mitmweb again

```bash
cd ~/apps/mitmproxy
./mitmweb --listen-host <PROXMOX_HOST_IP> --listen-port 8080
```

#### B. Re-set the global Android proxy in BlissOS

```bash
adb shell settings put global http_proxy <PROXMOX_HOST_IP>:8080
adb shell settings get global http_proxy
```

#### C. Open SSH tunnel again for the Web UI

```bash
ssh -L 8081:127.0.0.1:8081 user@<PROXMOX_HOST_IP>
```

#### D. Open the Web UI locally

```text
http://127.0.0.1:8081
```
