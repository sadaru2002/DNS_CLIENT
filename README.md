# Proxy DNS Service Implementation Guide

## Assignment 2 - System Administration
### Secure DNS Proxy with Unbound & DNSCrypt-Proxy

---

## 📖 Table of Contents

- [Introduction](#introduction)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Part 1: Debian Gateway Setup](#part-1-debian-internet-gateway-setup-computer-1)
- [Part 2: Windows Client Setup](#part-2-windows-client-setup-computer-2)
- [Part 3: Testing & Verification](#part-3-testing--verification)
- [Screenshot Guide](#screenshot-guide-for-report)
- [Troubleshooting](#troubleshooting)
- [Report Structure](#report-structure-suggestion)

---

## Introduction

This guide demonstrates the implementation of a secure Proxy DNS service on an Internet Gateway to obfuscate DNS traffic and prevent data theft through DNS traffic analysis.

**Technologies Used:**
- **Unbound DNS Server** - Local DNS resolver for client PCs
- **DNSCrypt-Proxy** - Encrypted upstream DNS proxy

**Operating Systems:**
- **Gateway (Computer 1):** Debian 12 (Bookworm)
- **Client (Computer 2):** Windows 10/11

---

## Problem Statement

The leadership team in a multinational IT company determined that data theft is occurring by analyzing DNS traffic at webservers through malware/bots/trojans. Even with TOR browser and TOR network, DNS traffic analysis remains a threat as TOR DNS servers may be compromised.

**Solution:** Implement a local DNS proxy service that:
1. Receives DNS queries from clients through Unbound DNS server
2. Encrypts and forwards queries through DNSCrypt-Proxy
3. Obfuscates DNS traffic to prevent analysis

---

## Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INTERNET                                    │
│                            │                                        │
│                    ┌───────▼───────┐                                │
│                    │  DNSCrypt     │  (Encrypted DNS Servers)       │
│                    │  Public DNS   │  (Cloudflare, Quad9, etc.)     │
│                    └───────▲───────┘                                │
│                            │ Encrypted Traffic (Port 443)           │
│  ┌─────────────────────────┼─────────────────────────────────────┐  │
│  │   DEBIAN INTERNET GATEWAY (Computer 1)                        │  │
│  │  ┌──────────────────────┴──────────────────────┐              │  │
│  │  │  DNSCrypt-Proxy (listening on 127.0.0.1:5353)│              │  │
│  │  └──────────────────────▲──────────────────────┘              │  │
│  │                         │ Local Forwarding                     │  │
│  │  ┌──────────────────────┴──────────────────────┐              │  │
│  │  │  Unbound DNS Server (listening on 0.0.0.0:53)│              │  │
│  │  └──────────────────────▲──────────────────────┘              │  │
│  └─────────────────────────┼─────────────────────────────────────┘  │
│                            │ Port 53 (Standard DNS)                 │
│  ┌─────────────────────────┼─────────────────────────────────────┐  │
│  │   WINDOWS CLIENT PC (Computer 2)                              │  │
│  │           DNS queries → Gateway IP                            │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### How It Works

1. **Windows Client PC** sends DNS query to Debian Internet Gateway (port 53)
2. **Unbound** receives the query and forwards to DNSCrypt-Proxy (port 5353)
3. **DNSCrypt-Proxy** encrypts the query and sends to secure DNS servers
4. Response travels back through the same encrypted path

---

## Prerequisites

| Computer | Role | Operating System |
|----------|------|------------------|
| **Computer 1** | Internet Gateway (DNS Server) | **Debian 12 (Bookworm)** |
| **Computer 2** | Client PC | **Windows 10/11** |

**Network Requirements:**
- Both computers connected to the same network
- Computer 1 (Debian) must have a static IP or known IP address
- VirtualBox with Guest Additions for clipboard sharing (if using VMs)

---

# Part 1: Debian Internet Gateway Setup (Computer 1)

## Step 1.1: Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

📸 **Screenshot 1:** Terminal showing successful update completion

---

## Step 1.2: Install Unbound DNS Server

```bash
sudo apt install unbound unbound-host -y
```

📸 **Screenshot 2:** Terminal showing Unbound installation

**Verify installation:**
```bash
unbound -V
```

📸 **Screenshot 3:** Unbound version output

---

## Step 1.3: Stop systemd-resolved (Important for Debian)

Debian may have systemd-resolved using port 53. Disable it:

```bash
# Check if systemd-resolved is running
sudo systemctl status systemd-resolved

# Stop and disable it
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

# Remove the symbolic link
sudo rm /etc/resolv.conf

# Create a new resolv.conf
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
```

📸 **Screenshot 4:** Terminal showing systemd-resolved disabled

---

## Step 1.4: Install DNSCrypt-Proxy

```bash
# Install required tools
sudo apt install wget curl -y

# Download the latest DNSCrypt-proxy
cd /tmp
wget https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.1.5/dnscrypt-proxy-linux_x86_64-2.1.5.tar.gz

# Extract
tar -xvzf dnscrypt-proxy-linux_x86_64-2.1.5.tar.gz

# Move to /opt
sudo mv linux-x86_64 /opt/dnscrypt-proxy

# Navigate to directory
cd /opt/dnscrypt-proxy
```

📸 **Screenshot 5:** Terminal showing DNSCrypt-proxy download and extraction

---

## Step 1.5: Configure DNSCrypt-Proxy

```bash
# Create configuration from example
sudo cp example-dnscrypt-proxy.toml dnscrypt-proxy.toml

# Edit configuration
sudo nano dnscrypt-proxy.toml
```

**Key configuration settings in `dnscrypt-proxy.toml`:**

```toml
# Listen on localhost port 5353 (to avoid conflict with Unbound on port 53)
listen_addresses = ['127.0.0.1:5353']

# Use secure DNS servers
server_names = ['cloudflare', 'google', 'quad9-dnscrypt-ip4-filter-pri']

# Enable IPv4
ipv4_servers = true

# Disable IPv6 (optional, depending on your network)
ipv6_servers = false

# Require DNSSEC
require_dnssec = true

# Require no-logging policy
require_nolog = true
```

📸 **Screenshot 6:** The edited dnscrypt-proxy.toml configuration file

---

## Step 1.6: Install DNSCrypt-Proxy as a System Service

```bash
# Install as service
sudo ./dnscrypt-proxy -service install

# Start the service
sudo ./dnscrypt-proxy -service start

# Check status
sudo ./dnscrypt-proxy -service status
```

📸 **Screenshot 7:** DNSCrypt-proxy service running status

**Test DNSCrypt is working:**
```bash
./dnscrypt-proxy -resolve google.com
```

📸 **Screenshot 8:** DNSCrypt-proxy resolving a domain successfully

---

## Step 1.7: Configure Unbound DNS Server

```bash
# Backup original config
sudo cp /etc/unbound/unbound.conf /etc/unbound/unbound.conf.backup

# Create new configuration
sudo nano /etc/unbound/unbound.conf
```

**Complete Unbound configuration for Debian:**

```yaml
server:
    # Network interface settings - listen on all interfaces
    interface: 0.0.0.0
    port: 53
    
    # Access control - allow local network (adjust to your network)
    access-control: 127.0.0.0/8 allow
    access-control: 192.168.0.0/16 allow
    access-control: 10.0.0.0/8 allow
    access-control: 172.16.0.0/12 allow
    
    # Performance settings
    num-threads: 2
    msg-cache-size: 64m
    rrset-cache-size: 128m
    
    # Privacy settings
    hide-identity: yes
    hide-version: yes
    
    # Security settings
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: yes
    
    # Logging (for demonstration)
    verbosity: 1
    log-queries: yes
    logfile: "/var/log/unbound/unbound.log"
    
    # Allow queries from localhost to DNSCrypt
    do-not-query-localhost: no

# Forward all queries to DNSCrypt-proxy
forward-zone:
    name: "."
    forward-addr: 127.0.0.1@5353
```

📸 **Screenshot 9:** The complete Unbound configuration file

---

## Step 1.8: Create Log Directory and Set Permissions

```bash
# Create log directory
sudo mkdir -p /var/log/unbound

# Set ownership
sudo chown unbound:unbound /var/log/unbound

# Set permissions
sudo chmod 755 /var/log/unbound
```

---

## Step 1.9: Validate Unbound Configuration

```bash
sudo unbound-checkconf
```

📸 **Screenshot 10:** Configuration validation showing "no errors"

---

## Step 1.10: Start Unbound Service

```bash
# Restart Unbound
sudo systemctl restart unbound

# Enable on boot
sudo systemctl enable unbound

# Check status
sudo systemctl status unbound
```

📸 **Screenshot 11:** Unbound service running successfully

---

## Step 1.11: Configure Firewall (iptables for Debian)

```bash
# Install iptables if not present
sudo apt install iptables -y

# Allow DNS traffic on port 53
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 53 -j ACCEPT

# Save iptables rules
sudo apt install iptables-persistent -y
sudo netfilter-persistent save

# View rules
sudo iptables -L -n
```

📸 **Screenshot 12:** Firewall rules showing DNS ports open

---

## Step 1.12: Get Gateway IP Address

```bash
ip addr show
# or
hostname -I
```

📸 **Screenshot 13:** Gateway's IP address (e.g., 192.168.1.100)

> ⚠️ **IMPORTANT: Note this IP address - you'll need it for the Windows client configuration!**

---

## Step 1.13: Test Local DNS Resolution

```bash
# Test with dig
dig @127.0.0.1 google.com

# Test with nslookup
nslookup google.com 127.0.0.1
```

📸 **Screenshot 14:** Successful local DNS resolution on Debian gateway

---

# Part 2: Windows Client Setup (Computer 2)

## Step 2.1: Open Network Settings

1. Press **Windows + I** to open Settings
2. Click **Network & Internet**
3. Click **Ethernet** (or **Wi-Fi** if using wireless)
4. Click on your active network connection

📸 **Screenshot 15:** Windows Network & Internet settings

---

## Step 2.2: Configure DNS Settings (Method 1 - Settings App)

### For Windows 11:

1. Click **Edit** next to DNS server assignment
2. Change from **Automatic (DHCP)** to **Manual**
3. Toggle **IPv4** to **On**
4. In **Preferred DNS**, enter the Debian Gateway IP (e.g., `192.168.1.100`)
5. Leave **Alternate DNS** empty or enter the same IP
6. Click **Save**

📸 **Screenshot 16:** Windows 11 DNS configuration with Gateway IP

---

### For Windows 10:

1. Click **Change adapter options**
2. Right-click your network adapter → **Properties**
3. Select **Internet Protocol Version 4 (TCP/IPv4)**
4. Click **Properties**
5. Select **Use the following DNS server addresses**
6. Enter the Debian Gateway IP (e.g., `192.168.1.100`)
7. Click **OK**

📸 **Screenshot 17:** Windows 10 TCP/IPv4 Properties with DNS configured

---

## Step 2.3: Configure DNS Settings (Method 2 - Control Panel)

1. Press **Windows + R**, type `ncpa.cpl`, press Enter
2. Right-click your network adapter → **Properties**
3. Select **Internet Protocol Version 4 (TCP/IPv4)**
4. Click **Properties**
5. Select **Use the following DNS server addresses**
6. **Preferred DNS server:** Enter Debian Gateway IP (e.g., `192.168.1.100`)
7. Click **OK** → **Close**

📸 **Screenshot 18:** Control Panel network adapter DNS configuration

---

## Step 2.4: Flush DNS Cache

Open **Command Prompt as Administrator** and run:

```cmd
ipconfig /flushdns
```

📸 **Screenshot 19:** DNS cache flushed successfully

---

## Step 2.5: Verify DNS Configuration

Open **Command Prompt** or **PowerShell** and run:

```powershell
# Check DNS server configuration
ipconfig /all
```

Look for the **DNS Servers** entry - it should show your Debian Gateway IP.

📸 **Screenshot 20:** ipconfig showing configured DNS server as Gateway IP

---

## Step 2.6: Alternative - PowerShell Method

Open **PowerShell as Administrator**:

```powershell
# View current DNS settings
Get-DnsClientServerAddress

# Set DNS server (replace "Ethernet" with your adapter name)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "192.168.1.100"

# Verify the change
Get-DnsClientServerAddress
```

📸 **Screenshot 21:** PowerShell showing DNS server configuration

---

# Part 3: Testing & Verification

## Step 3.1: Test DNS Resolution from Windows Client

Open **Command Prompt** on Windows and run:

```cmd
nslookup google.com
```

The server should show your Debian Gateway IP address.

📸 **Screenshot 22:** nslookup showing DNS resolution through Gateway

**Additional test:**

```cmd
nslookup facebook.com
nslookup github.com
```

📸 **Screenshot 23:** Multiple DNS queries resolved successfully

---

## Step 3.2: Test Internet Connectivity

```cmd
ping google.com
```

📸 **Screenshot 24:** Successful ping showing DNS resolution and connectivity

---

## Step 3.3: Verify Traffic on Debian Gateway

On the **Debian Gateway (Computer 1)**, monitor the Unbound logs:

```bash
# Watch Unbound logs in real-time
sudo tail -f /var/log/unbound/unbound.log
```

Now perform DNS queries from the **Windows Client** and watch the logs appear.

📸 **Screenshot 25:** Unbound log showing incoming queries from Windows client IP

---

## Step 3.4: Verify DNSCrypt Encryption

On the **Debian Gateway**, verify DNSCrypt is encrypting traffic:

```bash
# Check DNSCrypt-proxy is running
ps aux | grep dnscrypt

# Test resolution through DNSCrypt
cd /opt/dnscrypt-proxy
./dnscrypt-proxy -resolve microsoft.com
```

📸 **Screenshot 26:** DNSCrypt-proxy resolving domains with encryption info

---

## Step 3.5: Network Traffic Analysis (Demonstrates Encryption)

On **Debian Gateway**, install and use tcpdump:

```bash
# Install tcpdump
sudo apt install tcpdump -y

# Capture DNS traffic from Windows client (port 53)
sudo tcpdump -i any port 53 -n

# In another terminal, capture outgoing encrypted traffic (port 443)
sudo tcpdump -i any port 443 -n
```

📸 **Screenshot 27:** tcpdump showing DNS queries from Windows client
📸 **Screenshot 28:** tcpdump showing encrypted outbound traffic (no readable DNS)

---

## Step 3.6: Before/After Comparison

| Aspect | Traditional DNS | With Proxy DNS |
|--------|-----------------|----------------|
| **Query Visibility** | Plaintext, readable | Encrypted, obfuscated |
| **External Observation** | DNS queries visible | Only HTTPS traffic visible |
| **DNS Spoofing** | Vulnerable | Protected by DNSSEC |
| **Privacy** | Low | High |

📸 **Screenshot 29:** Side-by-side Wireshark comparison (optional)

---

## Step 3.7: Test from Windows Browser

On **Windows Client**, open a web browser and:
1. Visit `https://www.google.com`
2. Visit `https://www.github.com`
3. Visit any website of your choice

While browsing, watch the Unbound logs on Debian.

📸 **Screenshot 30:** Web browser on Windows with Unbound logs showing activity

---

# Screenshot Guide for Report

| # | Description | Where/Command |
|---|-------------|---------------|
| 1 | Debian system update | `sudo apt update && sudo apt upgrade -y` |
| 2 | Unbound installation | `sudo apt install unbound` |
| 3 | Unbound version | `unbound -V` |
| 4 | systemd-resolved disabled | `sudo systemctl status systemd-resolved` |
| 5 | DNSCrypt download | `wget` and `tar` commands |
| 6 | DNSCrypt config file | `nano dnscrypt-proxy.toml` |
| 7 | DNSCrypt service status | `./dnscrypt-proxy -service status` |
| 8 | DNSCrypt resolve test | `./dnscrypt-proxy -resolve google.com` |
| 9 | Unbound config file | `nano /etc/unbound/unbound.conf` |
| 10 | Config validation | `sudo unbound-checkconf` |
| 11 | Unbound service status | `sudo systemctl status unbound` |
| 12 | Firewall rules | `sudo iptables -L -n` |
| 13 | Gateway IP address | `ip addr show` |
| 14 | Local DNS test | `dig @127.0.0.1 google.com` |
| 15 | Windows Network Settings | Settings → Network & Internet |
| 16 | Windows 11 DNS config | DNS server assignment edit |
| 17 | Windows 10 DNS config | TCP/IPv4 Properties dialog |
| 18 | Control Panel DNS | Network adapter properties |
| 19 | DNS cache flush | `ipconfig /flushdns` |
| 20 | Windows ipconfig | `ipconfig /all` |
| 21 | PowerShell DNS | `Get-DnsClientServerAddress` |
| 22 | nslookup test | `nslookup google.com` |
| 23 | Multiple DNS queries | `nslookup` various domains |
| 24 | Ping test | `ping google.com` |
| 25 | Unbound logs | `tail -f /var/log/unbound/unbound.log` |
| 26 | DNSCrypt verification | `./dnscrypt-proxy -resolve` |
| 27 | tcpdump port 53 | Incoming DNS from Windows |
| 28 | tcpdump port 443 | Encrypted outbound traffic |
| 29 | Traffic comparison | Before/after Wireshark |
| 30 | Browser + logs | Windows browser with Debian logs |

---

# Troubleshooting

## Debian Gateway Issues

| Issue | Solution |
|-------|----------|
| Port 53 already in use | `sudo systemctl stop systemd-resolved && sudo systemctl disable systemd-resolved` |
| Unbound fails to start | Check config: `sudo unbound-checkconf` |
| DNSCrypt not starting | Check logs: `journalctl -xe` |
| Permission denied | Use `sudo` for all commands |

## Windows Client Issues

| Issue | Solution |
|-------|----------|
| DNS not resolving | Verify Gateway IP is correct, flush DNS with `ipconfig /flushdns` |
| Can't reach Gateway | Check both machines are on same network, ping Gateway IP |
| Changes not applying | Disable/enable network adapter, or restart Windows |
| DNS settings reverting | Disable automatic DNS assignment in DHCP |

## Network Issues

| Issue | Solution |
|-------|----------|
| Firewall blocking | On Debian: `sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT` |
| Connection timeout | Verify Gateway firewall allows port 53 |
| No internet on client | Verify Gateway has internet, check routing |

---

# Report Structure Suggestion

## 1. Title Page
- Assignment Title: Proxy DNS Service Implementation
- Course: System Administration
- Group Members: [Names and Student IDs]
- Date: [Submission Date]
- University: University of Sri Jayewardenepura

## 2. Introduction
- Problem statement (DNS traffic analysis/data theft)
- Proposed solution overview
- Objectives of the implementation

## 3. Literature Review
- What is DNS and why it's vulnerable
- Overview of Unbound DNS Server
- Overview of DNSCrypt-Proxy
- Benefits of encrypted DNS

## 4. Network Architecture
- Diagram showing component relationships
- Technology choices and justification
- Network topology (Debian Gateway + Windows Client)

## 5. Implementation

### 5.1 Debian Gateway Configuration
- Step-by-step installation
- Configuration files
- Service setup

### 5.2 Windows Client Configuration
- Network settings configuration
- DNS server assignment
- Verification steps

## 6. Testing & Verification
- DNS resolution tests
- Traffic analysis showing encryption
- Performance observations

## 7. Discussion
- Benefits of the implementation
- Security improvements achieved
- Limitations and challenges faced

## 8. Conclusion
- Summary of work done
- Future improvements
- Recommendations

## 9. References
- [Unbound Documentation](https://nlnetlabs.nl/documentation/unbound/)
- [DNSCrypt-Proxy GitHub](https://github.com/DNSCrypt/dnscrypt-proxy)
- Related academic papers and articles

---

# Security Benefits Achieved

✅ **DNS Query Encryption** - All outbound DNS queries are encrypted via DNSCrypt  
✅ **Traffic Obfuscation** - DNS traffic appears as regular HTTPS traffic (port 443)  
✅ **DNSSEC Validation** - Protection against DNS spoofing and cache poisoning  
✅ **Privacy Protection** - External DNS providers don't see individual client IPs  
✅ **Centralized Control** - All client DNS traffic routes through controlled gateway  
✅ **Logging Capability** - Complete audit trail of DNS queries for security monitoring  

---

# Quick Reference Commands

## Debian Gateway

```bash
# Check Unbound status
sudo systemctl status unbound

# View Unbound logs
sudo tail -f /var/log/unbound/unbound.log

# Check DNSCrypt status
cd /opt/dnscrypt-proxy && ./dnscrypt-proxy -service status

# Test DNS resolution
dig @127.0.0.1 google.com

# Restart services
sudo systemctl restart unbound
```

## Windows Client

```cmd
# Flush DNS cache
ipconfig /flushdns

# View DNS configuration
ipconfig /all

# Test DNS resolution
nslookup google.com

# Ping test
ping google.com
```

---

# Authors

**Group Assignment - System Administration**  
University of Sri Jayewardenepura

---

# License

This project is submitted as part of academic coursework.
