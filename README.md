# Proxy DNS Service Implementation Guide

## Assignment 2 - System Administration
### Secure DNS Proxy with Unbound & DNSCrypt-Proxy

**⚠️ IMPORTANT: Follow steps in exact order to avoid breaking internet connectivity!**

---

## Network Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INTERNET                                    │
│                            │                                        │
│                    ┌───────▼───────┐                                │
│                    │  DNSCrypt     │  (Encrypted DNS Servers)       │
│                    │  Public DNS   │                                │
│                    └───────▲───────┘                                │
│                            │ Encrypted (Port 443)                   │
│  ┌─────────────────────────┼─────────────────────────────────────┐  │
│  │   DEBIAN GATEWAY (Computer 1)                                 │  │
│  │       DNSCrypt-Proxy (127.0.0.1:5353)                         │  │
│  │              ▲                                                 │  │
│  │       Unbound DNS (0.0.0.0:53)                                │  │
│  └─────────────────────────┼─────────────────────────────────────┘  │
│                            │ Port 53                                │
│  ┌─────────────────────────┼─────────────────────────────────────┐  │
│  │   WINDOWS CLIENT (Computer 2)                                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

# PART 1: DEBIAN GATEWAY SETUP

## Prerequisites
- Fresh Debian 12 installation in VirtualBox
- Network: NAT or Bridged Adapter
- Internet working on VM

---

## STEP 1: Add User to Sudo Group (If Needed)

If you get "user not in sudoers" error:

```bash
su -
```

Enter root password, then:

```bash
usermod -aG sudo YOUR_USERNAME
```

```bash
exit
```

```bash
su - YOUR_USERNAME
```

📸 **Screenshot 1:** User added to sudo group

---

## STEP 2: Fix Package Sources

```bash
sudo nano /etc/apt/sources.list
```

Delete everything and paste this:

```
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
```

Save: **Ctrl+O** → **Enter** → **Ctrl+X**

📸 **Screenshot 2:** sources.list file

---

## STEP 3: Update System

```bash
sudo apt update
```

```bash
sudo apt upgrade -y
```

📸 **Screenshot 3:** System update complete

---

## STEP 4: Install Unbound DNS Server

```bash
sudo apt install unbound unbound-host -y
```

Verify:

```bash
unbound -V
```

📸 **Screenshot 4:** Unbound installed and version shown

---

## STEP 5: Install Required Tools

```bash
sudo apt install wget curl tar -y
```

---

## STEP 6: Download DNSCrypt-Proxy

```bash
cd /tmp
```

```bash
wget https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.1.5/dnscrypt-proxy-linux_x86_64-2.1.5.tar.gz
```

📸 **Screenshot 5:** DNSCrypt download complete

---

## STEP 7: Extract and Install DNSCrypt

```bash
tar -xvzf dnscrypt-proxy-linux_x86_64-2.1.5.tar.gz
```

```bash
sudo mv linux-x86_64 /opt/dnscrypt-proxy
```

```bash
cd /opt/dnscrypt-proxy
```

```bash
ls -la
```

📸 **Screenshot 6:** DNSCrypt files in /opt/dnscrypt-proxy

---

## STEP 8: Configure DNSCrypt-Proxy

```bash
sudo cp example-dnscrypt-proxy.toml dnscrypt-proxy.toml
```

```bash
sudo nano dnscrypt-proxy.toml
```

Find and change these lines:

```toml
listen_addresses = ['127.0.0.1:5353']
```

```toml
server_names = ['cloudflare', 'google']
```

```toml
ipv4_servers = true
```

```toml
require_dnssec = true
```

Save: **Ctrl+O** → **Enter** → **Ctrl+X**

📸 **Screenshot 7:** dnscrypt-proxy.toml configuration

---

## STEP 9: Start DNSCrypt-Proxy Service

```bash
sudo ./dnscrypt-proxy -service install
```

```bash
sudo ./dnscrypt-proxy -service start
```

📸 **Screenshot 8:** DNSCrypt service started

Verify it's running:

```bash
ps aux | grep dnscrypt
```

📸 **Screenshot 9:** DNSCrypt process running

---

## STEP 10: Test DNSCrypt

```bash
./dnscrypt-proxy -resolve google.com
```

📸 **Screenshot 10:** DNSCrypt resolving domain

---

## STEP 11: Configure Unbound

Backup original:

```bash
sudo cp /etc/unbound/unbound.conf /etc/unbound/unbound.conf.backup
```

Edit config:

```bash
sudo nano /etc/unbound/unbound.conf
```

Delete everything and paste this:

```yaml
server:
    interface: 0.0.0.0
    port: 53
    access-control: 127.0.0.0/8 allow
    access-control: 192.168.0.0/16 allow
    access-control: 10.0.0.0/8 allow
    do-not-query-localhost: no
    hide-identity: yes
    hide-version: yes

forward-zone:
    name: "."
    forward-addr: 127.0.0.1@5353
```

Save: **Ctrl+O** → **Enter** → **Ctrl+X**

📸 **Screenshot 11:** Unbound configuration file

---

## STEP 12: Validate Unbound Config

```bash
sudo unbound-checkconf
```

Should say: `unbound-checkconf: no errors`

📸 **Screenshot 12:** No errors in config

---

## STEP 13: Restart Unbound

```bash
sudo systemctl restart unbound
```

```bash
sudo systemctl enable unbound
```

```bash
sudo systemctl status unbound
```

📸 **Screenshot 13:** Unbound service running

---

## STEP 14: Test Local DNS

```bash
dig @127.0.0.1 google.com
```

📸 **Screenshot 14:** DNS query working through localhost

---

## STEP 15: Get Gateway IP Address

```bash
ip addr show
```

Note the IP address (example: 192.168.1.100)

📸 **Screenshot 15:** Gateway IP address

---

## STEP 16: Allow Firewall (Optional)

```bash
sudo apt install iptables -y
```

```bash
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
```

```bash
sudo iptables -A INPUT -p tcp --dport 53 -j ACCEPT
```

📸 **Screenshot 16:** Firewall rules added

---

# PART 2: WINDOWS CLIENT SETUP

---

## STEP 17: Open Network Settings

1. Press **Windows + I**
2. Click **Network & Internet**
3. Click **Ethernet** or **Wi-Fi**
4. Click your connection

📸 **Screenshot 17:** Windows Network Settings

---

## STEP 18: Configure DNS

### Windows 11:
1. Click **Edit** next to DNS
2. Change to **Manual**
3. Turn on **IPv4**
4. **Preferred DNS:** Enter Debian Gateway IP (e.g., 192.168.1.100)
5. Click **Save**

### Windows 10:
1. Click **Change adapter options**
2. Right-click adapter → **Properties**
3. Select **IPv4** → **Properties**
4. Select **Use the following DNS**
5. Enter Debian Gateway IP
6. Click **OK**

📸 **Screenshot 18:** DNS configured to Gateway IP

---

## STEP 19: Flush DNS Cache

Open **Command Prompt as Administrator**:

```cmd
ipconfig /flushdns
```

📸 **Screenshot 19:** DNS cache flushed

---

## STEP 20: Verify DNS Setting

```cmd
ipconfig /all
```

Look for **DNS Servers** - should show your Gateway IP.

📸 **Screenshot 20:** DNS server shows Gateway IP

---

# PART 3: TESTING

---

## STEP 21: Test from Windows

```cmd
nslookup google.com
```

Server should show your Debian Gateway IP.

📸 **Screenshot 21:** nslookup showing Gateway as DNS server

---

## STEP 22: Test Internet

```cmd
ping google.com
```

📸 **Screenshot 22:** Ping working

---

## STEP 23: Monitor on Debian

On Debian Gateway, watch queries come in:

```bash
sudo tcpdump -i any port 53 -n
```

Then make DNS queries from Windows.

📸 **Screenshot 23:** tcpdump showing Windows client queries

---

## STEP 24: Verify Encryption

On Debian, check external traffic is encrypted:

```bash
sudo tcpdump -i any port 443 -n
```

📸 **Screenshot 24:** Encrypted outbound traffic

---

# SCREENSHOT CHECKLIST

| # | Description | Command/Location |
|---|-------------|------------------|
| 1 | User added to sudo | `usermod` command |
| 2 | sources.list | `/etc/apt/sources.list` |
| 3 | System update | `apt update` output |
| 4 | Unbound version | `unbound -V` |
| 5 | DNSCrypt download | `wget` output |
| 6 | DNSCrypt files | `ls -la` in /opt/dnscrypt-proxy |
| 7 | DNSCrypt config | `dnscrypt-proxy.toml` |
| 8 | DNSCrypt started | `-service start` output |
| 9 | DNSCrypt process | `ps aux | grep dnscrypt` |
| 10 | DNSCrypt resolve | `-resolve google.com` |
| 11 | Unbound config | `/etc/unbound/unbound.conf` |
| 12 | Config validation | `unbound-checkconf` |
| 13 | Unbound status | `systemctl status unbound` |
| 14 | Local DNS test | `dig @127.0.0.1 google.com` |
| 15 | Gateway IP | `ip addr show` |
| 16 | Firewall rules | `iptables` commands |
| 17 | Windows Network | Settings app |
| 18 | Windows DNS config | DNS settings dialog |
| 19 | DNS flush | `ipconfig /flushdns` |
| 20 | Windows DNS verify | `ipconfig /all` |
| 21 | nslookup test | `nslookup google.com` |
| 22 | Ping test | `ping google.com` |
| 23 | tcpdump port 53 | Client queries visible |
| 24 | tcpdump port 443 | Encrypted traffic |

---

# TROUBLESHOOTING

| Problem | Solution |
|---------|----------|
| Not in sudoers | `su -` then `usermod -aG sudo username` |
| Package not found | Fix `/etc/apt/sources.list` and `apt update` |
| No internet | `echo "nameserver 8.8.8.8" \| sudo tee /etc/resolv.conf` |
| Port 53 in use | `sudo systemctl stop systemd-resolved` |
| Unbound won't start | `sudo unbound-checkconf` to find errors |
| Windows can't resolve | Check Gateway IP is correct, flush DNS |

---

# Authors

**System Administration Assignment**  
University of Sri Jayewardenepura

