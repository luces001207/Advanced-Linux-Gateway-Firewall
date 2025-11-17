# 🔐 Secure Linux Firewall & Network Proxy Gateway  
*A Cybersecurity Engineering Project Demonstrating Routing, NAT, DNS Security, Firewalling, Logging, and Traffic Analysis*

This project implements a **Linux-based secure gateway** that acts as a **router, firewall, DNS caching server, and traffic inspection proxy** for an isolated internal Windows/Linux client. All traffic from the internal network is forced through this hardened Linux gateway, allowing complete visibility and control over outbound and inbound packets.

---

## 📌 Table of Contents
- [Overview](#overview)  
- [Key Features](#key-features)  
- [Architecture](#architecture)  
- [Skills Demonstrated](#skills-demonstrated)  
- [Technical Implementation](#technical-implementation)  
  - [1. Network Configuration](#1-network-configuration)  
  - [2. DNS Caching Server (Bind9)](#2-dns-caching-server-bind9)  
  - [3. NAT & Routing](#3-nat--routing)  
  - [4. Firewall Rules](#4-firewall-rules)  
  - [5. Advanced Security Controls](#5-advanced-security-controls)  
  - [6. Traffic Capture & Analysis](#6-traffic-capture--analysis)  
- [Repository Structure](#repository-structure)  
- [How to Reproduce This Lab](#how-to-reproduce-this-lab)  
- [Future Improvements](#future-improvements)  
- [License](#license)

---

## Overview

This project configures a Linux virtual machine to operate as a **secure network gateway** with:

- Layer 3 routing  
- Network Address Translation (NAT)  
- DNS caching and inspection  
- Stateful packet filtering (iptables)  
- Traffic logging  
- Rate limiting  
- Selective ICMP control  
- Packet capture and analysis  

The internal VM has **no direct internet access**—all packets flow through the secure gateway, implementing network segmentation similar to a corporate perimeter firewall.

---

## Key Features

### 🔐 **Firewall + Router + Proxy**
The Linux VM simultaneously acts as:
- A router for the internal network  
- A NAT gateway to the internet  
- A firewall controlling allowed/blocked traffic  
- A DNS caching server  
- A packet monitoring node  

### 🛡 Hardened Firewall Policy
- Only specific outbound ports allowed  
- All inbound traffic is blocked by default  
- Logging of dropped packets  
- Protection against excessive connection attempts  

### 🚦 Rate Limiting  
HTTPS SYN packets are limited to **10/hour** to prevent abuse or scanning.

### 📡 ICMP Restrictions  
- Internal VM can send echo requests  
- But cannot receive replies  
- Prevents lateral network scanning  

### 📊 Traffic Monitoring  
Full packet capture to verify routing, filtering, and blocked packets.

---

## Architecture 

```
 ┌──────────────────────────────┐
 │       Internet (NAT)         │
 └───────────────┬──────────────┘
                eth0
        ┌───────────────────┐
        │  Linux Firewall   │
        │  Router + Proxy   │
        └───────────────────┘
                eth1
 ┌──────────────────────────────┐
 │ Internal Windows/Linux Client│
 │     10.0.100.0/24 Network    │
 └──────────────────────────────┘
```

The internal VM uses:
- Gateway: **10.0.100.1**
- DNS Resolver: **10.0.100.1**
- Everything passes through the firewall VM.

---

## Skills Demonstrated

### 🛡 Cybersecurity & System Hardening
- Designing firewall rule sets  
- Implementing least privilege access  
- Traffic inspection & monitoring  

### 🌐 Networking  
- Routing & subnets  
- NAT/IP masquerading  
- DNS caching & upstream forwarding  

### 🧰 Tools  
- Ubuntu Linux  
- iptables  
- Bind9  
- tcpdump  
- netplan  
- VirtualBox  

---

## Technical Implementation

## 1. Network Configuration

### Gateway Interface Setup

```
eth0 → NAT  
eth1 → Internal Network (10.0.100.1/24)
```

Netplan Example:
```yaml
network:
  version: 2
  ethernets:
    eth1:
      addresses: [10.0.100.1/24]
```

Enable routing:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

---

## 2. DNS Caching Server (Bind9)

Bind9 configured as a caching resolver with Google DNS upstream:

```
forwarders {
    8.8.8.8;
};
```

Internal VM queries → gateway → Bind9 → cached for speed.

---

## 3. NAT & Routing

Allow traffic from eth1 → eth0:
```bash
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
```

Protect against unsolicited inbound traffic:
```bash
iptables -A FORWARD -i eth0 -o eth1   -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Enable NAT:
```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

---

## 4. Firewall Rules

### Allowed Traffic
| Service | Port | Direction | Reason |
|---------|------|-----------|--------|
| DNS | 53/udp | Out | Resolve domains |
| HTTPS | 443/tcp | Out | Secure browsing |
| SSH | 10020/tcp | Out | Controlled remote access |

Rules:
```bash
iptables -A FORWARD -i eth1 -o eth0 -p tcp --dport 443 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -p udp --dport 53 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -p tcp --dport 10020 -j ACCEPT
```

### Block Everything Else
```bash
iptables -A FORWARD -j DROP
```

---

## 5. Advanced Security Controls

### Logging Dropped Packets
```bash
iptables -A FORWARD -j LOG   --log-prefix "lmong lab4 dropped: " --log-level 4
```

Logs stored in:
```
/var/log/syslog
```

---

### HTTPS SYN Rate Limiting
```bash
iptables -A FORWARD -p tcp --syn --dport 443   -m limit --limit 10/hour --limit-burst 10 -j ACCEPT
```

---

### ICMP Filtering

Allow only outgoing pings:
```bash
iptables -A FORWARD -i eth1 -o eth0   -p icmp --icmp-type echo-request -j ACCEPT
```

Block all incoming responses:
```bash
iptables -A FORWARD -i eth0 -o eth1 -p icmp -j DROP
```

---

# Repository Structure 

```
.
├── README.md
├── config/
│   ├── netplan/
│   ├── bind9/
│   ├── iptables/
├── captures/
│   ├── dns_test.pcap
│   ├── https_limit.pcap
│   ├── icmp_blocked.pcap
│   └── ssh_test.pcap
├── logs/
│   └── firewall.log
└── docs/
    ├── architecture.png
    └── firewall_policy.md
```
---

## Future Improvements

- Migrate firewall from iptables → nftables  
- Add Suricata IDS/IPS  
- Add Snort signature-based detection  
- Deploy Squid proxy server  
- Automate setup with Ansible  

---

## License

For educational and ethical cybersecurity use only.
