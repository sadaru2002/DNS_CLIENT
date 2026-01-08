# Proxy DNS Service Implementation Guide

## Assignment 2 - System Administration
### Secure DNS Proxy with Unbound & DNSCrypt-Proxy

---

## 📖 Table of Contents

- [Introduction](#introduction)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Part 1: Internet Gateway Setup](#part-1-internet-gateway-setup-computer-1)
- [Part 2: Client PC Setup](#part-2-client-pc-setup-computer-2)
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
│  │     INTERNET GATEWAY (Computer 1)                             │  │
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
│  │     CLIENT PC (Computer 2)                                    │  │
│  │           DNS queries → Gateway IP                            │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### How It Works

1. **Client PC** sends DNS query to Internet Gateway (port 53)
2. **Unbound** receives the query and forwards to DNSCrypt-Proxy (port 5353)
3. **DNSCrypt-Proxy** encrypts the query and sends to secure DNS servers
4. Response travels back through the same encrypted path

---

## Prerequisites

| Computer | Role | OS Recommendation |
|----------|------|-------------------|
| **Computer 1** | Internet Gateway (DNS Server) | Ubuntu 22.04 LTS / Debian 12 |
| **Computer 2** | Client PC | Any Linux/Windows |

**Network Requirements:**
- Both computers connected to the same network
- Computer 1 must have a static IP or known IP address

---

## Part 1: Internet Gateway Setup (Computer 1)

### Step 1.1: Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

📸 **Screenshot 1:** Terminal showing successful update completion

---

### Step 1.2: Install Unbound DNS Server

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

### Step 1.3: Install DNSCrypt-Proxy

```bash
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

📸 **Screenshot 4:** Terminal showing DNSCrypt-proxy download and extraction

---

### Step 1.4: Configure DNSCrypt-Proxy

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

📸 **Screenshot 5:** The edited dnscrypt-proxy.toml configuration file

---

### Step 1.5: Install DNSCrypt-Proxy as a System Service

```bash
# Install as service
sudo ./dnscrypt-proxy -service install

# Start the service
sudo ./dnscrypt-proxy -service start

# Check status
sudo ./dnscrypt-proxy -service status
```

📸 **Screenshot 6:** DNSCrypt-proxy service running status

**Test DNSCrypt is working:**
```bash
./dnscrypt-proxy -resolve google.com
```

📸 **Screenshot 7:** DNSCrypt-proxy resolving a domain successfully

---

### Step 1.6: Configure Unbound DNS Server

```bash
# Backup original config
sudo cp /etc/unbound/unbound.conf /etc/unbound/unbound.conf.backup

# Create new configuration
sudo nano /etc/unbound/unbound.conf
```

**Complete Unbound configuration:**

```yaml
server:
    # Network interface settings
    interface: 0.0.0.0
    port: 53
    
    # Access control - allow local network
    access-control: 127.0.0.0/8 allow
    access-control: 192.168.0.0/16 allow
    access-control: 10.0.0.0/8 allow
    
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
    
    # Don't use system resolv.conf
    do-not-query-localhost: no

# Forward all queries to DNSCrypt-proxy
forward-zone:
    name: "."
    forward-addr: 127.0.0.1@5353
```

📸 **Screenshot 8:** The complete Unbound configuration file

---

### Step 1.7: Create Log Directory and Set Permissions

```bash
# Create log directory
sudo mkdir -p /var/log/unbound

# Set ownership
sudo chown unbound:unbound /var/log/unbound
```

---

### Step 1.8: Validate Unbound Configuration

```bash
sudo unbound-checkconf
```

📸 **Screenshot 9:** Configuration validation showing "no errors"

---

### Step 1.9: Start Unbound Service

```bash
# Restart Unbound
sudo systemctl restart unbound

# Enable on boot
sudo systemctl enable unbound

# Check status
sudo systemctl status unbound
```

📸 **Screenshot 10:** Unbound service running successfully

---

### Step 1.10: Configure Firewall (if enabled)

```bash
# Allow DNS traffic
sudo ufw allow 53/udp
sudo ufw allow 53/tcp

# Check firewall status
sudo ufw status
```

📸 **Screenshot 11:** Firewall rules showing DNS ports open

---

### Step 1.11: Get Gateway IP Address

```bash
ip addr show
# or
hostname -I
```

📸 **Screenshot 12:** Gateway's IP address (e.g., 192.168.1.100)

> ⚠️ **Note this IP address - you'll need it for the client configuration!**

---

## Part 2: Client PC Setup (Computer 2)

### Step 2.1: Configure DNS Settings

#### On Linux (Ubuntu/Debian):

**Method 1: Using Netplan (Ubuntu 18.04+)**
```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eth0:                          # Replace with your interface name
      dhcp4: true
      nameservers:
        addresses: [192.168.1.100]  # Replace with Gateway IP
```

```bash
sudo netplan apply
```

**Method 2: Direct resolv.conf**
```bash
sudo nano /etc/resolv.conf
```

```
nameserver 192.168.1.100   # Replace with Gateway IP
```

📸 **Screenshot 13:** Client's DNS configuration file

---

#### On Windows:

1. Open **Control Panel** → **Network and Internet** → **Network and Sharing Center**
2. Click on your active connection
3. Click **Properties**
4. Select **Internet Protocol Version 4 (TCP/IPv4)**
5. Click **Properties**
6. Select **Use the following DNS server addresses**
7. Enter the Gateway IP (e.g., 192.168.1.100)
8. Click **OK**

📸 **Screenshot 14:** Windows DNS configuration dialog

---

### Step 2.2: Verify DNS Configuration

**On Linux:**
```bash
cat /etc/resolv.conf
```

**On Windows (PowerShell):**
```powershell
Get-DnsClientServerAddress
```

📸 **Screenshot 15:** Client showing configured DNS server

---

## Part 3: Testing & Verification

### Step 3.1: Test DNS Resolution from Client

**On Linux:**
```bash
# Test basic resolution
nslookup google.com

# Test with dig
dig google.com

# Test with host
host google.com
```

**On Windows:**
```powershell
nslookup google.com
```

📸 **Screenshot 16:** Successful DNS resolution from client showing Gateway IP as DNS server

---

### Step 3.2: Verify Traffic Flow on Gateway

On the **Gateway (Computer 1)**, monitor the Unbound logs:

```bash
# Watch Unbound logs in real-time
sudo tail -f /var/log/unbound/unbound.log
```

Now perform DNS queries from the Client PC and watch the logs.

📸 **Screenshot 17:** Unbound log showing incoming queries from client

---

### Step 3.3: Verify DNSCrypt Encryption

On the **Gateway**, check DNSCrypt-proxy logs:

```bash
# Check DNSCrypt logs
sudo journalctl -u dnscrypt-proxy -f
# OR
cat /opt/dnscrypt-proxy/dnscrypt-proxy.log
```

📸 **Screenshot 18:** DNSCrypt-proxy logs showing encrypted DNS traffic

---

### Step 3.4: Network Traffic Analysis (Demonstration)

Install Wireshark or tcpdump to capture and analyze traffic:

```bash
# Install tcpdump
sudo apt install tcpdump -y

# Capture DNS traffic on port 53 (internal - between client and gateway)
sudo tcpdump -i any port 53 -n

# Capture external traffic on port 443 (encrypted DNSCrypt traffic)
sudo tcpdump -i any port 443 -n
```

📸 **Screenshot 19:** tcpdump showing local DNS queries from client
📸 **Screenshot 20:** tcpdump showing encrypted outbound traffic (no readable DNS data)

---

### Step 3.5: Before/After Comparison

| Traditional DNS | With Proxy DNS |
|-----------------|----------------|
| Queries sent in plaintext | Queries encrypted via DNSCrypt |
| DNS data visible to network observers | DNS data obfuscated |
| Vulnerable to DNS spoofing | Protected by DNSSEC |

📸 **Screenshot 21:** Side-by-side comparison showing encrypted vs unencrypted DNS traffic

---

## Screenshot Guide for Report

| Screenshot # | Description | Command/Location |
|--------------|-------------|------------------|
| 1 | System update completion | `sudo apt update && sudo apt upgrade -y` |
| 2 | Unbound installation | `sudo apt install unbound` |
| 3 | Unbound version | `unbound -V` |
| 4 | DNSCrypt download | `wget` and `tar` commands |
| 5 | DNSCrypt config file | `nano dnscrypt-proxy.toml` |
| 6 | DNSCrypt service status | `./dnscrypt-proxy -service status` |
| 7 | DNSCrypt resolve test | `./dnscrypt-proxy -resolve google.com` |
| 8 | Unbound config file | `nano /etc/unbound/unbound.conf` |
| 9 | Config validation | `sudo unbound-checkconf` |
| 10 | Unbound service status | `sudo systemctl status unbound` |
| 11 | Firewall rules | `sudo ufw status` |
| 12 | Gateway IP address | `ip addr show` |
| 13 | Client DNS config (Linux) | `cat /etc/resolv.conf` |
| 14 | Client DNS config (Windows) | Network adapter properties |
| 15 | DNS configuration verification | `Get-DnsClientServerAddress` |
| 16 | DNS resolution test | `nslookup google.com` |
| 17 | Unbound logs | `tail -f /var/log/unbound/unbound.log` |
| 18 | DNSCrypt logs | DNSCrypt log file |
| 19 | Local DNS traffic capture | `tcpdump port 53` |
| 20 | Encrypted outbound traffic | `tcpdump port 443` |
| 21 | Traffic comparison | Before/after Wireshark capture |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Port 53 already in use | `sudo systemctl stop systemd-resolved` and `sudo systemctl disable systemd-resolved` |
| Unbound fails to start | Check configuration with `sudo unbound-checkconf` |
| Client can't resolve DNS | Verify firewall allows port 53: `sudo ufw allow 53` |
| DNSCrypt not encrypting | Check `listen_addresses` is set to `127.0.0.1:5353` |
| Connection timeout | Verify both computers are on the same network |
| Permission denied errors | Run commands with `sudo` |

---

## Report Structure Suggestion

### 1. Title Page
- Assignment Title: Proxy DNS Service Implementation
- Course: System Administration
- Group Members: [Names and IDs]
- Date: [Submission Date]

### 2. Introduction
- Problem statement (DNS traffic analysis/data theft)
- Proposed solution overview
- Objectives

### 3. Literature Review
- What is DNS and why it's vulnerable
- Overview of Unbound DNS Server
- Overview of DNSCrypt-Proxy
- Benefits of encrypted DNS

### 4. Network Architecture
- Diagram showing component relationships
- Technology choices and justification
- Network topology

### 5. Implementation
- Step-by-step installation and configuration
- Screenshots at each major step
- Configuration file explanations

### 6. Testing & Verification
- DNS resolution tests
- Traffic analysis showing encryption
- Performance comparison

### 7. Discussion
- Benefits of the implementation
- Security improvements achieved
- Limitations

### 8. Conclusion
- Summary of work done
- Future improvements

### 9. References
- [Unbound Documentation](https://nlnetlabs.nl/documentation/unbound/)
- [DNSCrypt-Proxy GitHub](https://github.com/DNSCrypt/dnscrypt-proxy)
- Related academic papers

---

## Security Benefits Achieved

✅ **DNS Query Encryption** - All outbound DNS queries are encrypted  
✅ **Traffic Obfuscation** - DNS traffic appears as regular HTTPS traffic  
✅ **DNSSEC Validation** - Protection against DNS spoofing  
✅ **Privacy Protection** - DNS providers don't see client IPs  
✅ **Centralized Control** - All DNS traffic routes through gateway  

---

## Authors

**Group Assignment - System Administration**  
University of Sri Jayewardenepura

---

## License

This project is submitted as part of academic coursework.

