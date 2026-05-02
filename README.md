# Operational Reference: Network Reconnaissance & Evasion

**Classification:** RESTRICTED — FOR AUTHORIZED USE ONLY

**Operational Unit:** Red Cell — Technical Reconnaissance Division

**Author/Source:** Compiled from field notes and operational guidance; includes excerpts from "My Nmap Cheat Sheet" (Dan Fedele, Aug 2023), HackTricks, and suicidal_teddy.

---

## Purpose

This repository is a consolidated operational reference for network reconnaissance, scanning techniques, and firewall/IDS interaction. Intended for audits, lawful penetration tests, red-team operations, and defensive teams seeking to understand how reconnaissance tools interact with network enforcement points. All material is provided for **authorized, legal, and ethical use only**.

Operators must ensure:

* A signed Rules of Engagement (RoE) or equivalent authorization is in place.
* The scope of activity is explicitly defined and approved.
* All local laws and organizational policies are followed.

---

## Table of Contents

* [Summary](#summary)
* [Firewall/IDS Evasion — Source Ports](#best-source-ports-for-firewallids-evasion)
* [Recommended Command Style](#recommended-nmap-command-style)
* [Rationale: Why `--source-port` Works](#rationale-why---source-port-can-affect-enforcement)
* [Ports NOT Recommended](#ports-not-recommended)
* [Top 3 for Evasion](#top-3-for-operational-evasion)
* [Stealth Combination](#stealth-combination)
* [Advanced Evasion Techniques](#advanced-evasion-techniques)
* [Nmap Cheat Sheet — Operational Notes](#nmap-cheat-sheet--operational-notes)
  * [Typical Usage (Ippsec Special)](#typical-usage-ippsec-special)
  * [Port Scanning Techniques](#unique-methods--port-scanning-techniques)
  * [Scan All TCP Ports](#scan-all-tcp-ports)
  * [Scan Specific TCP Ports](#scan-specific-tcp-ports)
  * [UDP Scans](#udp-scans)
  * [Host Discovery](#host-discovery-ping-sweep-an-entire-subnet)
  * [Useful Flags](#useful-flags)
  * [Proxy Usage](#nmap-through-a-proxy)
* [Target Specification](#target-specification)
* [Timing Templates](#timing-templates-in-detail)
* [Version Detection](#version-detection-intensity)
* [OS Fingerprinting](#os-fingerprinting)
* [Firewall Detection & Mapping](#firewall-detection--mapping)
* [NSE Scripts: Usage and Guidance](#nmap-scripts-usage-and-guidance)
  * [HTTP](#http-scripts)
  * [SMB](#smb-scripts)
  * [DNS](#dns-scripts)
  * [SNMP](#snmp-scripts)
  * [MSSQL](#mssql-scripts)
  * [MySQL](#mysql-scripts)
  * [RDP](#rdp-scripts)
  * [VNC](#vnc-scripts)
  * [FTP](#ftp-scripts)
  * [SSH](#ssh-scripts)
  * [SMTP](#smtp-scripts)
  * [LDAP](#ldap-enumeration)
  * [SSL/TLS](#ssltls-scripts)
* [IPv6 Scanning](#ipv6-scanning)
* [Output Formats & Parsing](#output-formats--parsing)
* [Performance Tuning](#performance-tuning)
* [Tool Integration](#tool-integration)
* [Engagement Methodology](#real-world-engagement-methodology)
* [Quick Reference Combos](#quick-reference-combos)
* [Operational Guidance & Defensive Considerations](#operational-guidance--defensive-considerations)
* [Appendix: Credits](#appendix-raw-notes--credits)

---

## Summary

Networks and enforcement appliances are occasionally misconfigured or rely on legacy assumptions about service behavior. In operational assessments, understanding which source ports and scan techniques are more likely to pass through permissive or misconfigured enforcement is useful for both offensive and defensive teams. This document compiles commonly observed source ports, command patterns, and a comprehensive nmap reference.

---

## Best Source Ports for Firewall/IDS Evasion

These source ports are commonly whitelisted, treated as trusted, or otherwise subject to less strict inspection due to historical service expectations or misconfiguration:

| # | Port | Service | Rationale |
|---|------|---------|-----------|
| 1 | 53 | DNS | Frequently allowed for both inbound and outbound DNS traffic |
| 2 | 20/21 | FTP Active/Control | Some stateless or legacy firewalls treat FTP traffic as trusted |
| 3 | 80 | HTTP | Often allowed where web traffic is permitted; source port 80 less inspected |
| 4 | 443 | HTTPS | Similar to 80; in many deployments subjected to even less inspection |
| 5 | 123 | NTP | Some networks permit NTP broadly; IDS may not thoroughly inspect |
| 6 | 179 | BGP | Older routing equipment may implicitly trust BGP ports |
| 7 | 500/4500 | IPsec | Corporate firewalls supporting IPsec may whitelist these |
| 8 | 67/68 | DHCP | Often trusted or ignored on internal networks |
| 9 | 88 | Kerberos | Trusted in AD environments; often allowed through domain firewalls |

The presence of these ports reflects operational observation of misconfigurations and permissive policies; it is not a guarantee they will succeed in any environment.

---

## Recommended Nmap Command Style

```bash
sudo nmap -Pn --source-port 53 -f 10.20.30.40
```

```bash
sudo nmap -Pn -f --mtu 16 --source-port 53 -D RND:5 10.20.30.40
```

These examples combine source-port selection and packet fragmentation and are included as case studies of common command constructions found in operational playbooks.

---

## Rationale: Why `--source-port` Can Affect Enforcement

Some enforcement systems make legacy assumptions about traffic semantics (for instance: "traffic from port 53 must be DNS"), which can lead to overly permissive rules. Using a particular source port may cause a firewall or ACL to treat probe traffic differently, enabling reconnaissance in environments with such misconfigurations.

Such techniques can interact with:

* Stateless packet filters
* Misconfigured cloud security groups
* ACL-based vendor appliances (legacy Cisco, MikroTik, SonicWall, etc.)
* NAT devices with lax source-port translation rules

---

## Ports NOT Recommended

Avoid using the following unless operationally required and authorized:

* **0** — Often dropped or explicitly logged by enforcement devices.
* **High ephemeral ranges (49152+)** — Typical of normal outbound traffic; unlikely to evade inspection.
* **Well-monitored service ports** (SSH/22, SMTP/25, POP/110, IMAP/143) — Frequently subject to logging and active monitoring.

---

## Top 3 for Operational Evasion

| Rank | Port | Protocol | Why |
|------|------|----------|-----|
| 1 | 53 | DNS | Nearly universally allowed; both UDP and TCP |
| 2 | 80 | HTTP | Outbound web traffic is baseline-expected |
| 3 | 443 | HTTPS | Often receives even less DPI than port 80 |

Success is environment-dependent.

---

## Stealth Combination

Pattern combining fragmentation, trusted source port, and decoys:

```bash
sudo nmap -Pn -f --mtu 16 --source-port 53 -D RND:5 10.20.30.40
```

---

## Advanced Evasion Techniques

### TTL Manipulation

```bash
# Set custom TTL to evade firewalls that check TTL values
sudo nmap --ttl 64 -sS 192.168.1.0/24

# Match TTL to appear as traffic from a specific OS (Linux=64, Windows=128, Cisco=255)
sudo nmap --ttl 128 -sS 10.0.0.1
```

### Data Length Padding

```bash
# Append random data to reach specific packet size (evades length-based IDS signatures)
sudo nmap --data-length 24 -sS 192.168.1.1

# Append custom hex payload
sudo nmap --data "\xde\xad\xbe\xef" -sS 192.168.1.1

# Append custom ASCII string
sudo nmap --data-string "GET / HTTP/1.0" -sS 192.168.1.1
```

### MAC Address Spoofing

```bash
# Spoof MAC to a specific address
sudo nmap --spoof-mac 00:11:22:33:44:55 -sS 192.168.1.0/24

# Use a random MAC from a specific vendor (bypasses NAC)
sudo nmap --spoof-mac Apple -sS 192.168.1.0/24
sudo nmap --spoof-mac Dell -sS 192.168.1.0/24
sudo nmap --spoof-mac Cisco -sS 192.168.1.0/24

# Completely random MAC
sudo nmap --spoof-mac 0 -sS 192.168.1.0/24
```

### IP Options Manipulation

```bash
# Record Route (trace path through network)
sudo nmap --ip-options "R" -sS 192.168.1.1

# Loose source routing (specify intermediate hops)
sudo nmap --ip-options "L 192.168.1.1 10.0.0.1" -sS 172.16.0.1

# Strict source routing
sudo nmap --ip-options "S 192.168.1.1 10.0.0.1" -sS 172.16.0.1

# Timestamp option
sudo nmap --ip-options "T" -sS 192.168.1.1
```

### Fragmentation Techniques

```bash
# Fragment packets into 8-byte fragments
sudo nmap -f -sS 192.168.1.1

# Double fragmentation (16-byte fragments)
sudo nmap -f -f -sS 192.168.1.1

# Set specific MTU (must be multiple of 8)
sudo nmap --mtu 16 -sS 192.168.1.1
sudo nmap --mtu 24 -sS 192.168.1.1

# Combined fragmentation + evasion
sudo nmap -f --data-length 100 --ttl 64 -D RND:5 -sS 192.168.1.1
```

### Decoy Scanning

```bash
# Random decoys (ME specifies your real position in the decoy list)
sudo nmap -D RND:10,ME -sS 192.168.1.1

# Specific decoy IPs
sudo nmap -D 10.0.0.1,10.0.0.2,10.0.0.3,ME,10.0.0.5 -sS 192.168.1.1

# Combined with source port manipulation
sudo nmap -D RND:5 -g 53 -sS 192.168.1.1
```

### Idle (Zombie) Scan — Ultimate Stealth

```bash
# Use another host as a zombie (completely blind scan)
sudo nmap -sI zombie_host:80 192.168.1.1

# Find suitable zombie hosts (need predictable IP ID sequence)
sudo nmap -O -v 192.168.1.0/24 | grep "IP ID Sequence"
```

### Timing-Based Evasion

```bash
# Paranoid: 5 minute gap between probes
sudo nmap -T0 -sS 192.168.1.1

# Sneaky: 15 second gap between probes
sudo nmap -T1 -sS 192.168.1.1

# Custom scan delay (one probe every 10 seconds)
sudo nmap --scan-delay 10s -sS 192.168.1.1

# Rate limiting to stay under IDS thresholds
sudo nmap --max-rate 10 -sS 192.168.1.0/24

# Randomize target order to avoid sequential detection
sudo nmap -sS --randomize-hosts 192.168.1.0/24
```

### Custom TCP Flags

```bash
# SYN+FIN (can confuse some firewalls)
sudo nmap --scanflags SYNFIN -p 80 192.168.1.1

# All flags set
sudo nmap --scanflags URGACKPSHRSTSYNFIN -p 80 192.168.1.1
```

### Proxy Chains

```bash
# Route through HTTP/SOCKS4 proxies (connect scan only)
sudo nmap --proxies socks4://proxy1:1080,http://proxy2:8080 -sT 192.168.1.1
```

### Combined Stealth Scan Examples

```bash
# Maximum evasion single target
sudo nmap -sS -T1 -f --data-length 24 -g 53 --spoof-mac 0 -D RND:10 -p- target.com

# Internal network stealth
sudo nmap -sS -T2 --source-port 53 -f --randomize-hosts 192.168.1.0/24

# Pass through strict firewall
sudo nmap -Pn -sS --source-port 443 --mtu 24 --data-length 50 -T2 -D RND:3 10.0.0.1
```

---

## Nmap Cheat Sheet — Operational Notes

Nmap is a fundamental reconnaissance tool for network assessments. The notes below collect typical command patterns, flags, and considerations used by experienced operators.

### Typical Usage (Ippsec Special)

```bash
sudo nmap -sC -sV -oN portscan -v 10.20.30.40
```

* `-sC` runs default scripts
* `-sV` attempts service/version detection
* `-oN` saves normal output to a file
* `-v` increases verbosity

Running as root (via `sudo`) enables SYN scans and other low-level techniques that are restricted to privileged users.

---

### Unique Methods — Port Scanning Techniques

| Flag | Name | Description |
|------|------|-------------|
| `-sS` | SYN scan | Sends SYN without completing handshake. Requires privilege. Stealthier than connect scan; detectable by modern IDS. |
| `-sT` | TCP connect scan | Full three-way handshake. No special privileges needed but generates connection logs. |
| `-sU` | UDP scan | Probes UDP services. States: open (reply), closed (ICMP unreachable), filtered, open\|filtered (no response). Use `-sV` for protocol probes. |
| `-sY` | SCTP INIT scan | Probes SCTP endpoints. In some deployments INIT failures don't leave host logs. |
| `-sN` | Null scan | No TCP flags set. Relies on RFC-compliant stacks: RST=closed, silence=open\|filtered. Unreliable on Windows. |
| `-sF` | FIN scan | FIN flag only. Same interpretation as null scan. |
| `-sX` | Xmas scan | FIN+PSH+URG flags. Same interpretation as null/FIN scans. |
| `-sM` | Maimon scan | FIN+ACK. Historically effective against BSD; most modern stacks treat as closed. |
| `-sA` | ACK scan | For firewall/ACL mapping. Unfiltered=RST received, filtered=no response or ICMP error. |
| `-sW` | Window scan | Like ACK but inspects RST window values to distinguish open/closed. Heuristic, platform-dependent. |
| `-sI <zombie>` | Idle scan | Uses zombie host with predictable IPID to infer port states while hiding scanner origin. |
| `--badsum` | Bad checksum | Invalid checksums. Endhosts discard; middleboxes may respond — reveals firewall behavior. |
| `-sZ` | SCTP COOKIE-ECHO | Sends COOKIE-ECHO fragments. May pass middleboxes. Often can't distinguish open vs filtered. |
| `-sO` | IP protocol scan | Scans IP protocol numbers (ICMP=1, TCP=6, UDP=17) rather than ports. |
| `-b <server>` | FTP bounce | Instructs FTP server to proxy connections. Most modern FTP servers disable this. |

### Scan All TCP Ports

```bash
sudo nmap -p- -oN allports -v 10.20.30.40
```

### Scan Specific TCP Ports

```bash
sudo nmap -sC -sV -p 22,25,80,443,5900-5999 -oN portscan 10.20.30.40
```

### UDP Scans

```bash
# Top 1000 UDP ports
sudo nmap -sU -oN udp-portscan -v 10.20.30.40

# All UDP ports (very slow)
sudo nmap -sU -p- -oN udp-all -v 10.20.30.40

# Combined TCP + UDP on common ports
sudo nmap -sS -sU -p T:1-65535,U:53,67,68,69,111,123,135,137,138,139,161,162,445,500,514,520,631,1434,1900,4500,49152 10.20.30.40
```

Note: UDP scans are inherently slow. Use `--top-ports` or specific port lists to keep times reasonable.

### Bypass Some IDS and Firewalls (Illustrative)

```bash
sudo nmap -Pn --source-port=9090 -f 10.20.30.40
```

### RING ALL THE BELLS

```bash
sudo nmap -p- -T5 --max-retries 0 -v -oA allports --script="default,vuln" 10.20.30.40
```

Maximum enumeration when stealth is irrelevant. Extreme aggression (`-T5`), no retransmission limit, default + vuln scripts on all ports.

### Host Discovery: Ping-sweep an Entire Subnet

```bash
sudo nmap -sn 10.20.30.0/24
```

ICMP may be blocked. Nmap uses several discovery techniques; behavior depends on privileges and network topology.

### Advanced Host Discovery

```bash
# ARP discovery (layer 2, local subnet only — most reliable)
sudo nmap -sn -PR 192.168.1.0/24

# TCP SYN discovery to specific ports
sudo nmap -sn -PS22,80,443,445,3389 192.168.1.0/24

# TCP ACK discovery (may pass stateless firewalls)
sudo nmap -sn -PA80,443 192.168.1.0/24

# Combine SYN + ACK for maximum coverage
sudo nmap -sn -PS80,443,22 -PA80,443,22 192.168.1.0/24

# UDP discovery probes
sudo nmap -sn -PU53,161,631 192.168.1.0/24

# SCTP INIT discovery (good for telecom networks)
sudo nmap -sn -PY2905,5060 192.168.1.0/24

# ICMP variants
sudo nmap -sn -PE 192.168.1.0/24           # Echo request
sudo nmap -sn -PP 192.168.1.0/24           # Timestamp request
sudo nmap -sn -PM 192.168.1.0/24           # Netmask request

# IP protocol ping
sudo nmap -sn -PO1,2,4 192.168.1.0/24

# Maximum discovery (throw everything)
sudo nmap -sn -PE -PP -PM -PS21,22,23,25,80,113,443,445,3389 -PA80,443 -PU53,111,137,161 192.168.1.0/24

# Broadcast-based discovery scripts
nmap --script broadcast-ping
nmap --script broadcast-dhcp-discover
nmap --script broadcast-netbios-master-browser
nmap --script broadcast-upnp-info
```

### VMware NAT Caveat

When running in a VM with NAT networking (VMware default), `-sn` may report all hosts as up because the NAT gateway responds on behalf of non-existent hosts.

```bash
# Mitigation: use --unprivileged
sudo nmap -sn --unprivileged 10.20.30.0/24
```

Nmap's `-sn` host-discovery probes:
```
ICMP ECHO_REQUEST -> ICMP reply
ICMP TIMESTAMP_REQUEST -> ICMP reply
443/TCP SYN -> SYN/ACK reply
80/TCP ACK -> RST reply
```

### Host Discovery: List Scan

```bash
sudo nmap -sL 10.20.30.0/24
```

Performs DNS reverse lookups only — covert, does not actively probe hosts.

### Useful Flags

| Flag | Purpose |
|------|---------|
| `-O` | OS detection |
| `-A` | Aggressive (`-O` + `-sV` + `-sC` + `--traceroute`) |
| `-T 0..5` | Timing template (0=paranoid, 5=insane) |
| `--min-rate` / `--max-rate` | Rate control (packets/sec) |
| `-Pn` | Skip host discovery (treat all hosts as online) |
| `-n` | Never DNS resolve |
| `-R` | Always DNS resolve (even for down hosts) |
| `--dns-servers` | Use custom DNS servers |
| `--open` | Only show open ports |
| `--reason` | Show reason for port state |
| `--packet-trace` | Show all packets sent/received |
| `--traceroute` | Trace path to each host |
| `--iflist` | Show interfaces and routes |
| `-r` | Don't randomize port order |
| `--stats-every 10s` | Show scan progress at interval |
| `-d[1-9]` | Debug output (increasing verbosity) |
| `-oN` / `-oA` / `-oG` / `-oX` | Output formats: normal / all / grepable / XML |

### Nmap Through a Proxy

```bash
# Native SOCKS/HTTP proxy support (connect scan only)
sudo nmap -sT --proxy socks4://40.40.50.50:5789 10.20.30.40

# Via proxychains
sudo proxychains nmap -sT 10.20.30.40
```

Note: Proxy support only works with `-sT` (connect scan). SYN scans and raw packet techniques cannot be proxied.

---

## Target Specification

### Input Methods

```bash
# Single IP
nmap 192.168.1.1

# CIDR notation
nmap 192.168.1.0/24

# Range
nmap 192.168.1.1-254

# Octet range
nmap 192.168.0-255.1-254

# Multiple targets mixed
nmap 192.168.1.0/24 10.0.0.1 172.16.0.0/16

# From file (one target per line)
nmap -iL targets.txt

# Random targets (research/testing)
nmap -iR 100 -sS -p 80
```

### Exclusions

```bash
# Exclude specific hosts
nmap 192.168.1.0/24 --exclude 192.168.1.1,192.168.1.254

# Exclude from file
nmap 192.168.1.0/24 --excludefile exclusions.txt
```

### Port Specification

```bash
# Mixed TCP and UDP
nmap -sS -sU -p T:80,443,U:53,161 192.168.1.1

# Top N ports (by frequency)
nmap --top-ports 100 192.168.1.0/24

# Port ratio (ports with frequency above threshold)
nmap --port-ratio 0.01 192.168.1.1

# All ports including port 0
nmap -p 0-65535 192.168.1.1

# SCTP ports
nmap -p S:2905,5060 -sY 192.168.1.1
```

---

## Timing Templates in Detail

| Template | Name | Scan Delay | Parallelism | Max RTT | Retries | Use Case |
|----------|------|-----------|-------------|---------|---------|----------|
| `-T0` | Paranoid | 5 min | Serial | — | 10 | IDS evasion, extremely stealthy |
| `-T1` | Sneaky | 15 sec | Serial | — | 10 | IDS evasion, slightly faster |
| `-T2` | Polite | 400ms | Serial | — | 10 | Reduce network load |
| `-T3` | Normal | None | Dynamic | — | 10 | Standard (default) |
| `-T4` | Aggressive | 10ms max | Dynamic | 1250ms | 6 | Fast reliable networks |
| `-T5` | Insane | 5ms max | Dynamic | 300ms | 2 | Lab/CTF, sacrifices accuracy |

### Overriding Individual Timing Parameters

```bash
# Override specific T4 settings
sudo nmap -T4 --max-rtt-timeout 200ms --max-retries 3 -sS 192.168.1.0/24

# Custom scan delay with max cap
sudo nmap --scan-delay 5s --max-scan-delay 20s -sS 192.168.1.1
```

---

## Version Detection Intensity

```bash
# Intensity 0: minimal probes (rarity 1 only) — fastest
sudo nmap -sV --version-intensity 0 192.168.1.1

# Light (intensity 2): common services
sudo nmap -sV --version-light 192.168.1.1

# Default (intensity 7): standard -sV
sudo nmap -sV 192.168.1.1

# All (intensity 9): try every probe — slowest, most thorough
sudo nmap -sV --version-all 192.168.1.1

# Debug version detection
sudo nmap -sV --version-trace -p 80 192.168.1.1
```

**When to use:**
- Quick sweep: `--version-light`
- Standard engagement: default `-sV`
- Stubborn/unusual services: `--version-all`
- Deep single-target: `--version-all --script=banner`

---

## OS Fingerprinting

```bash
# Standard OS detection
sudo nmap -O 192.168.1.1

# Aggressive guess when no exact match
sudo nmap -O --osscan-guess 192.168.1.1

# Limit to promising targets (need 1 open + 1 closed port)
sudo nmap -O --osscan-limit 192.168.1.0/24

# Maximum effort
sudo nmap -O --osscan-guess --max-os-tries 5 192.168.1.1

# Combined with version detection for better accuracy
sudo nmap -O -sV 192.168.1.1

# SMB-based Windows version detection (more accurate)
sudo nmap -p 445 --script smb-os-discovery 192.168.1.1

# SNMP-based OS identification
sudo nmap -sU -p 161 --script snmp-sysdescr 192.168.1.1
```

**TTL-based OS guessing (heuristic):**
| Default TTL | Operating System |
|-------------|-----------------|
| 64 | Linux / macOS / FreeBSD |
| 128 | Windows |
| 255 | Cisco IOS / Solaris |

---

## Firewall Detection & Mapping

### ACK Scan for Rule Mapping

```bash
# Determine which ports are filtered (firewall) vs unfiltered
sudo nmap -sA -p 1-1024 192.168.1.1
# unfiltered = no firewall (RST received)
# filtered = firewall dropping packets

# Window scan — can distinguish open vs closed behind firewall
sudo nmap -sW -p 1-1024 192.168.1.1
```

### Comparing Scan Types to Detect Firewalls

```bash
# SYN scan vs ACK scan — ports that differ indicate stateful firewall
sudo nmap -sS -p 1-1024 192.168.1.1 -oG syn_scan.gnmap
sudo nmap -sA -p 1-1024 192.168.1.1 -oG ack_scan.gnmap
```

### Firewalk and Bypass Scripts

```bash
# Determine which ports the firewall allows through
sudo nmap --script firewalk --traceroute -p 1-100 192.168.1.1

# Firewall bypass attempt
sudo nmap --script firewall-bypass 192.168.1.1
```

### Diagnosing Firewall Behavior

```bash
# Show reason for every port state
sudo nmap --reason -sS -p 80,443 192.168.1.1

# Full packet trace for analysis
sudo nmap --packet-trace -sS -p 80 192.168.1.1

# TCP timestamp analysis (detect load balancers)
sudo nmap -p 80 --script=tcp-timestamps 192.168.1.1
```

---

## Nmap Scripts: Usage and Guidance

Nmap Scripting Engine (NSE) provides 600+ scripts for enumeration and vulnerability checks.

### NSE Categories

| Category | Description |
|----------|-------------|
| `auth` | Authentication bypass/handling |
| `broadcast` | Network broadcast discovery |
| `brute` | Brute force attacks |
| `default` | Scripts run with `-sC` |
| `discovery` | Active network discovery |
| `dos` | Denial of service testing |
| `exploit` | Active exploitation |
| `external` | Third-party service lookups |
| `fuzzer` | Fuzzing tests |
| `intrusive` | May crash/disrupt targets |
| `malware` | Malware detection |
| `safe` | Won't crash services |
| `version` | Version detection extensions |
| `vuln` | Vulnerability detection |

### Script Selection Syntax

```bash
# Run all scripts in a category
nmap --script vuln 192.168.1.1

# Boolean combinations
nmap --script "default and safe" 192.168.1.1
nmap --script "(http-* or ssl-*) and not http-slowloris" -p 80,443 192.168.1.1
nmap --script "not brute and not dos" 192.168.1.1

# Wildcard
nmap --script "smb-*" -p 445 192.168.1.1

# Script arguments
nmap --script http-brute --script-args 'userdb=users.txt,passdb=pass.txt' 192.168.1.1

# Script timeout
nmap --script-timeout 60s --script vuln 192.168.1.1

# Custom user-agent
nmap --script http-enum --script-args http.useragent="Mozilla/5.0" 192.168.1.1
```

### Script Database Management

```bash
# Update script database after adding new scripts
nmap --script-updatedb

# Show help for a script
nmap --script-help http-enum

# Trace script execution (debugging)
nmap --script-trace --script http-enum -p 80 192.168.1.1
```

---

### HTTP Scripts

```bash
# Comprehensive enumeration
sudo nmap -p 80,443,8080,8443 --script http-enum 192.168.1.1

# Title grabbing (mass recon)
sudo nmap -p 80 --script http-title 192.168.1.0/24

# HTTP methods (PUT, DELETE, TRACE)
sudo nmap -p 80 --script http-methods --script-args http-methods.url-path='/api' 192.168.1.1

# WAF detection
sudo nmap -p 80 --script http-waf-detect,http-waf-fingerprint 192.168.1.1

# Shellshock
sudo nmap -p 80 --script http-shellshock --script-args uri=/cgi-bin/status 192.168.1.1

# WordPress enumeration
sudo nmap -p 80 --script http-wordpress-enum,http-wordpress-users 192.168.1.1

# Git repo exposure
sudo nmap -p 80 --script http-git 192.168.1.1

# Headers and security headers
sudo nmap -p 80 --script http-headers,http-security-headers 192.168.1.1

# Virtual host discovery
sudo nmap -p 80 --script http-vhosts 192.168.1.1

# Default credentials check
sudo nmap -p 80 --script http-default-accounts 192.168.1.1

# Robots.txt
sudo nmap -p 80 --script http-robots.txt 192.168.1.1

# SQL injection testing
sudo nmap -p 80 --script http-sql-injection 192.168.1.1

# NTLM info disclosure (IIS)
sudo nmap -p 80 --script http-ntlm-info 192.168.1.1

# Sitemap generation
sudo nmap -p 80 --script http-sitemap-generator 192.168.1.1

# Slowloris vulnerability check
sudo nmap -p 80 --script http-slowloris-check 192.168.1.1
```

### SMB Scripts

```bash
# Full SMB enumeration
sudo nmap -p 445 --script smb-enum-shares,smb-enum-users,smb-enum-groups,smb-enum-domains,smb-enum-sessions 192.168.1.1

# OS discovery via SMB
sudo nmap -p 445 --script smb-os-discovery 192.168.1.1

# Protocol version and security mode
sudo nmap -p 445 --script smb-protocols,smb-security-mode 192.168.1.1

# EternalBlue (MS17-010)
sudo nmap -p 445 --script smb-vuln-ms17-010 192.168.1.1

# All SMB vulnerabilities
sudo nmap -p 445 --script "smb-vuln-*" 192.168.1.1

# SMB brute force
sudo nmap -p 445 --script smb-brute --script-args userdb=users.txt,passdb=passwords.txt 192.168.1.1

# List share contents
sudo nmap -p 445 --script smb-ls --script-args smb-ls.share=C$ 192.168.1.1
```

### DNS Scripts

```bash
# Zone transfer attempt
sudo nmap -p 53 --script dns-zone-transfer --script-args dns-zone-transfer.domain=target.com 192.168.1.1

# DNS brute force subdomains
sudo nmap -p 53 --script dns-brute --script-args dns-brute.domain=target.com 192.168.1.1

# Cache snooping
sudo nmap -p 53 --script dns-cache-snoop --script-args dns-cache-snoop.domains=facebook.com,gmail.com 192.168.1.1

# Check for open recursion
sudo nmap -p 53 --script dns-recursion 192.168.1.1

# SRV record enumeration
sudo nmap -p 53 --script dns-srv-enum --script-args dns-srv-enum.domain=target.com 192.168.1.1

# NSEC walking (enumerate DNSSEC zones)
sudo nmap -p 53 --script dns-nsec-enum --script-args dns-nsec-enum.domain=target.com 192.168.1.1

# Service discovery via DNS
sudo nmap -p 53 --script dns-service-discovery 192.168.1.1
```

### SNMP Scripts

```bash
# Full SNMP enumeration
sudo nmap -sU -p 161 --script snmp-info,snmp-interfaces,snmp-netstat,snmp-processes,snmp-sysdescr 192.168.1.1

# Windows-specific SNMP enumeration
sudo nmap -sU -p 161 --script snmp-win32-services,snmp-win32-shares,snmp-win32-software,snmp-win32-users 192.168.1.1

# SNMP community string brute force
sudo nmap -sU -p 161 --script snmp-brute --script-args snmp-brute.communitiesdb=communities.txt 192.168.1.1

# Cisco IOS config via SNMP
sudo nmap -sU -p 161 --script snmp-ios-config 192.168.1.1
```

### MSSQL Scripts

```bash
# Info gathering
sudo nmap -p 1433 --script ms-sql-info 192.168.1.1

# Empty/default password check
sudo nmap -p 1433 --script ms-sql-empty-password 192.168.1.1

# NTLM info (domain, hostname)
sudo nmap -p 1433 --script ms-sql-ntlm-info 192.168.1.1

# Dump password hashes (requires creds)
sudo nmap -p 1433 --script ms-sql-dump-hashes --script-args mssql.username=sa,mssql.password=password 192.168.1.1

# xp_cmdshell command execution (requires creds)
sudo nmap -p 1433 --script ms-sql-xp-cmdshell --script-args mssql.username=sa,mssql.password=password,ms-sql-xp-cmdshell.cmd="whoami" 192.168.1.1

# Brute force
sudo nmap -p 1433 --script ms-sql-brute --script-args userdb=users.txt,passdb=pass.txt 192.168.1.1

# Broadcast instance discovery
nmap --script broadcast-ms-sql-discover
```

### MySQL Scripts

```bash
# MySQL info and version
sudo nmap -p 3306 --script mysql-info 192.168.1.1

# Empty password check
sudo nmap -p 3306 --script mysql-empty-password 192.168.1.1

# User enumeration
sudo nmap -p 3306 --script mysql-enum 192.168.1.1

# List databases (requires creds)
sudo nmap -p 3306 --script mysql-databases --script-args mysqluser=root,mysqlpass='' 192.168.1.1

# Dump hashes
sudo nmap -p 3306 --script mysql-dump-hashes --script-args username=root,password='' 192.168.1.1

# Auth bypass CVE-2012-2122
sudo nmap -p 3306 --script mysql-vuln-cve2012-2122 192.168.1.1

# Brute force
sudo nmap -p 3306 --script mysql-brute 192.168.1.1
```

### RDP Scripts

```bash
# Encryption level enumeration
sudo nmap -p 3389 --script rdp-enum-encryption 192.168.1.1

# NTLM info from RDP (domain, hostname, OS version)
sudo nmap -p 3389 --script rdp-ntlm-info 192.168.1.1

# MS12-020 vulnerability check
sudo nmap -p 3389 --script rdp-vuln-ms12-020 192.168.1.1
```

### VNC Scripts

```bash
# VNC info (protocol version, auth types)
sudo nmap -p 5900 --script vnc-info 192.168.1.1

# VNC brute force
sudo nmap -p 5900 --script vnc-brute 192.168.1.1

# VNC title
sudo nmap -p 5900 --script vnc-title 192.168.1.1
```

### FTP Scripts

```bash
# Anonymous login check
sudo nmap -p 21 --script ftp-anon 192.168.1.1

# FTP brute force
sudo nmap -p 21 --script ftp-brute 192.168.1.1

# FTP bounce check
sudo nmap -p 21 --script ftp-bounce 192.168.1.1

# System type
sudo nmap -p 21 --script ftp-syst 192.168.1.1

# vsftpd 2.3.4 backdoor
sudo nmap -p 21 --script ftp-vsftpd-backdoor 192.168.1.1

# ProFTPD backdoor
sudo nmap -p 21 --script ftp-proftpd-backdoor 192.168.1.1
```

### SSH Scripts

```bash
# Host key retrieval
sudo nmap -p 22 --script ssh-hostkey --script-args ssh_hostkey=full 192.168.1.1

# Authentication methods
sudo nmap -p 22 --script ssh-auth-methods --script-args ssh.user=root 192.168.1.1

# Brute force
sudo nmap -p 22 --script ssh-brute --script-args userdb=users.txt,passdb=pass.txt 192.168.1.1

# Public key acceptance check
sudo nmap -p 22 --script ssh-publickey-acceptance --script-args ssh.publickey=id_rsa.pub 192.168.1.1

# Remote command execution (requires creds)
sudo nmap -p 22 --script ssh-run --script-args ssh-run.username=admin,ssh-run.password=pass,ssh-run.cmd="id" 192.168.1.1
```

### SMTP Scripts

```bash
# Commands enumeration
sudo nmap -p 25 --script smtp-commands 192.168.1.1

# User enumeration (VRFY/EXPN/RCPT)
sudo nmap -p 25 --script smtp-enum-users --script-args smtp-enum-users.methods={VRFY,EXPN,RCPT} 192.168.1.1

# Open relay check
sudo nmap -p 25 --script smtp-open-relay 192.168.1.1

# NTLM info (Exchange)
sudo nmap -p 25 --script smtp-ntlm-info 192.168.1.1

# Brute force
sudo nmap -p 25 --script smtp-brute 192.168.1.1
```

### LDAP Enumeration

```bash
# Root DSE information
sudo nmap -p 389 --script ldap-rootdse 192.168.1.1

# LDAP search
sudo nmap -p 389 --script ldap-search --script-args 'ldap.username="",ldap.password=""' 192.168.1.1

# Brute force
sudo nmap -p 389 --script ldap-brute --script-args ldap.base='"dc=target,dc=com"' 192.168.1.1
```

### SSL/TLS Scripts

```bash
# Enumerate cipher suites
sudo nmap -p 443 --script ssl-enum-ciphers 192.168.1.1

# Certificate information
sudo nmap -p 443 --script ssl-cert 192.168.1.1

# Heartbleed check (CVE-2014-0160)
sudo nmap -p 443 --script ssl-heartbleed 192.168.1.1

# POODLE check
sudo nmap -p 443 --script ssl-poodle 192.168.1.1

# CCS injection
sudo nmap -p 443 --script ssl-ccs-injection 192.168.1.1

# Known keys check
sudo nmap -p 443 --script ssl-known-key 192.168.1.1

# DH parameter check
sudo nmap -p 443 --script ssl-dh-params 192.168.1.1

# Combined TLS audit
sudo nmap -p 443,8443 --script "ssl-*" 192.168.1.1
```

### Vulnerability Integration (Vulners)

```bash
# CVE lookup for discovered services via vulners.com
sudo nmap -sV --script vulners 192.168.1.1

# Maximum coverage
sudo nmap -sV --version-all --script vulners -p- 192.168.1.1
```

---

## IPv6 Scanning

```bash
# Basic IPv6 scan
sudo nmap -6 -sS fe80::1

# IPv6 link-local (must specify interface)
sudo nmap -6 -sS fe80::1%eth0

# Full scan on IPv6 target
sudo nmap -6 -A -p- 2001:db8::1

# IPv6 neighbor discovery (multicast echo)
nmap -6 --script targets-ipv6-multicast-echo --script-args newtargets -sn

# MLD (Multicast Listener Discovery)
nmap -6 --script targets-ipv6-multicast-mld --script-args newtargets -sn

# SLAAC discovery
nmap -6 --script targets-ipv6-multicast-slaac --script-args newtargets -sn

# IPv4 to IPv6 mapping
nmap -6 --script targets-ipv6-map4to6 --script-args targets-ipv6-map4to6.IPv4Hosts=192.168.1.0/24,newtargets -sn

# EUI-64 address generation from MAC
nmap -6 --script targets-ipv6-eui64 --script-args newtargets,targets-ipv6-eui64.mac=00:11:22:33:44:55 -sn

# Node info query (hostname, IPv4 of IPv6 host)
nmap -6 --script ipv6-node-info fe80::1

# Reverse DNS sweeping for IPv6
nmap -6 --script dns-ip6-arpa-scan --script-args dns-ip6-arpa-scan.domain=target.com
```

---

## Output Formats & Parsing

### Formats

```bash
# All three major formats at once
nmap -oA scan_results -sV 192.168.1.0/24
# Creates: scan_results.nmap, scan_results.xml, scan_results.gnmap

# Normal output
nmap -oN scan.txt -sS 192.168.1.0/24

# XML (for automation and import)
nmap -oX scan.xml -sV 192.168.1.0/24

# Grepable (one host per line)
nmap -oG scan.gnmap -sS 192.168.1.0/24

# Append to existing output
nmap --append-output -oN ongoing_scan.txt -sS 192.168.1.0/24

# Resume interrupted scan
nmap --resume scan.gnmap
```

### Parsing Tricks

```bash
# Extract live hosts from discovery
nmap -sn 192.168.1.0/24 -oG - | grep "Up" | awk '{print $2}'

# Hosts with a specific port open
grep "80/open" scan.gnmap | awk '{print $2}'

# All open ports for one host as comma-separated list
nmap -p- --open 192.168.1.1 -oG - | grep -oP '\d+/open' | cut -d'/' -f1 | tr '\n' ',' | sed 's/,$//'

# Pipe grepable to stdout directly
nmap -sS -p 80,443 192.168.1.0/24 -oG - | grep open

# XML to HTML report
xsltproc scan.xml -o scan.html

# XML parsing with xmlstarlet
xmlstarlet sel -t -m "//port[state/@state='open']" -v "../../../address/@addr" -o ":" -v "@portid" -o "/" -v "service/@name" -n scan.xml
```

---

## Performance Tuning

### Parallelism

```bash
# Set minimum parallel probes (aggressive)
nmap --min-parallelism 100 -sS 192.168.1.0/24

# Cap maximum parallelism (stability)
nmap --max-parallelism 10 -sS 192.168.1.0/24

# Host group sizes (how many hosts scanned simultaneously)
nmap --min-hostgroup 64 --max-hostgroup 256 -sS 10.0.0.0/16
```

### Timeout Tuning

```bash
# Aggressive timeouts for fast networks
nmap --max-rtt-timeout 100ms --initial-rtt-timeout 50ms -sS 192.168.1.0/24

# Give up on slow hosts
nmap --host-timeout 5m -sS 10.0.0.0/16
```

### Rate Control

```bash
# Minimum rate (ensure at least N packets/sec)
nmap --min-rate 1000 -sS 192.168.1.0/24

# Maximum rate (cap for stealth or stability)
nmap --max-rate 100 -sS 192.168.1.0/24
```

### Retry Control

```bash
# Reduce retries for speed
nmap --max-retries 1 -sS 192.168.1.0/24

# Zero retries (fastest, least reliable)
nmap --max-retries 0 -sS 192.168.1.0/24
```

### Performance Profiles

```bash
# "Speed demon" — fast internal network
sudo nmap -T4 --min-rate 5000 --max-retries 1 --max-rtt-timeout 100ms --min-hostgroup 128 -sS -p- 192.168.1.0/24

# "Gentle giant" — avoid overloading target
sudo nmap -T2 --max-rate 50 --max-parallelism 5 --scan-delay 1s -sS 192.168.1.0/24

# "Large network" — optimized for /8 and larger
sudo nmap -T4 --min-hostgroup 256 --min-parallelism 64 --max-retries 2 --host-timeout 10m -sS 10.0.0.0/8
```

---

## Tool Integration

### Nmap + Masscan (Speed + Accuracy)

```bash
# Phase 1: Fast port discovery with masscan
masscan -p1-65535 192.168.1.0/24 --rate=10000 -oL masscan_results.txt

# Phase 2: Extract ports and feed to nmap for service detection
ports=$(grep "open" masscan_results.txt | awk '{print $3}' | sort -un | tr '\n' ',' | sed 's/,$//')
hosts=$(grep "open" masscan_results.txt | awk '{print $4}' | sort -u)
sudo nmap -sV -sC -p "$ports" $hosts -oA detailed_scan
```

### Nmap + Metasploit

```bash
# Generate XML for Metasploit import
sudo nmap -sV -sC -oX scan.xml 192.168.1.0/24
# In msfconsole: db_import scan.xml
```

### Nmap + Searchsploit

```bash
# Scan and search for exploits based on versions
sudo nmap -sV -oX scan.xml 192.168.1.1 && searchsploit --nmap scan.xml
```

### Nmap + Nikto

```bash
# Feed discovered web servers to nikto
nmap -p 80,443,8080,8443 --open -oG web.gnmap 192.168.1.0/24
grep "open" web.gnmap | awk '{print $2}' | while read host; do nikto -h "$host"; done
```

### Nmap + CrackMapExec/NetExec

```bash
# Find SMB hosts then pass to crackmapexec
nmap -p 445 --open -oG smb.gnmap 192.168.1.0/24
grep "445/open" smb.gnmap | awk '{print $2}' | netexec smb - -u '' -p ''
```

### Nmap + Aquatone/Eyewitness

```bash
# Generate web service list for screenshot tools
nmap -p 80,443,8080,8443 --open -oX web.xml 192.168.1.0/24
cat web.xml | aquatone -nmap
```

---

## Real-World Engagement Methodology

### Phase 1: Network Discovery

```bash
# Quick ping sweep
sudo nmap -sn -PE -PP -PM -PS80,443,22,445 -PA80,443 -T4 10.0.0.0/16 -oA discovery

# Extract live hosts
grep "Up" discovery.gnmap | awk '{print $2}' > live_hosts.txt
```

### Phase 2: Fast Port Discovery

```bash
# Quick top-ports on all live hosts
sudo nmap -sS -T4 --top-ports 1000 --open -iL live_hosts.txt -oA quick_ports

# Full port scan
sudo nmap -sS -T4 -p- --open --min-rate 5000 -iL live_hosts.txt -oA all_ports
```

### Phase 3: Service Enumeration

```bash
# Version detection on discovered ports
ports=$(grep -oP '\d+/open' all_ports.gnmap | cut -d'/' -f1 | sort -un | tr '\n' ',' | sed 's/,$//')
sudo nmap -sV -sC -p "$ports" -iL live_hosts.txt -oA service_enum

# UDP (run in parallel)
sudo nmap -sU -T4 --top-ports 50 -iL live_hosts.txt -oA udp_scan
```

### Phase 4: Targeted Deep Scans

```bash
# Web servers
grep -E "80/open|443/open|8080/open" service_enum.gnmap | awk '{print $2}' > web_hosts.txt
sudo nmap -p 80,443,8080,8443 --script "http-enum,http-headers,http-methods,http-title,http-git,http-robots.txt,http-waf-detect" -iL web_hosts.txt -oA web_deep

# SMB hosts
grep "445/open" service_enum.gnmap | awk '{print $2}' > smb_hosts.txt
sudo nmap -p 445 --script "smb-enum-shares,smb-enum-users,smb-os-discovery,smb-security-mode,smb-vuln-*" -iL smb_hosts.txt -oA smb_deep

# Databases
grep -E "1433/open|3306/open|5432/open" service_enum.gnmap | awk '{print $2}' > db_hosts.txt
sudo nmap -p 1433,3306,5432 --script "ms-sql-info,ms-sql-empty-password,mysql-info,mysql-empty-password" -iL db_hosts.txt -oA db_deep
```

### Phase 5: Vulnerability Assessment

```bash
# Broad vuln scan
sudo nmap --script vuln -iL live_hosts.txt -oA vuln_scan

# Specific high-value checks
sudo nmap -p 445 --script smb-vuln-ms17-010 -iL smb_hosts.txt -oA eternalblue
sudo nmap -p 443,8443 --script ssl-enum-ciphers,ssl-cert,ssl-heartbleed -iL web_hosts.txt -oA ssl_audit
```

### Automated Pipeline Script

```bash
#!/bin/bash
TARGET_NET="${1:-192.168.1.0/24}"
OUTDIR="./scan_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$OUTDIR"

echo "[*] Phase 1: Host Discovery"
nmap -sn -T4 -PE -PS80,443,22,445 "$TARGET_NET" -oA "$OUTDIR/01_discovery"
grep "Up" "$OUTDIR/01_discovery.gnmap" | awk '{print $2}' > "$OUTDIR/live_hosts.txt"
echo "[+] Found $(wc -l < "$OUTDIR/live_hosts.txt") live hosts"

echo "[*] Phase 2: Port Discovery"
nmap -sS -T4 -p- --open --min-rate 3000 -iL "$OUTDIR/live_hosts.txt" -oA "$OUTDIR/02_all_ports"

echo "[*] Phase 3: Service Enumeration"
PORTS=$(grep -oP '\d+/open' "$OUTDIR/02_all_ports.gnmap" | cut -d'/' -f1 | sort -un | tr '\n' ',' | sed 's/,$//')
nmap -sV -sC -p "$PORTS" -iL "$OUTDIR/live_hosts.txt" -oA "$OUTDIR/03_services"

echo "[*] Phase 4: Vulnerability Scan"
nmap --script vuln -p "$PORTS" -iL "$OUTDIR/live_hosts.txt" -oA "$OUTDIR/04_vulns"

echo "[*] Phase 5: UDP Top 50"
nmap -sU -T4 --top-ports 50 -iL "$OUTDIR/live_hosts.txt" -oA "$OUTDIR/05_udp"

echo "[+] Complete. Results in $OUTDIR/"
```

---

## Quick Reference Combos

```bash
# "Just got on the network" — immediate situational awareness
sudo nmap -sn -PR 192.168.1.0/24 && sudo nmap -sS -T4 --top-ports 20 --open 192.168.1.0/24

# "Find low-hanging fruit fast"
sudo nmap -sV --script "default and safe and vuln" -T4 --open -iL targets.txt

# "Stealth scan a single high-value target"
sudo nmap -sS -T1 -f --data-length 24 -g 53 --spoof-mac 0 -D RND:10 -p- target.com

# "Comprehensive single-host assessment"
sudo nmap -sS -sU -sV -O -A --script "default,vuln,discovery" -p- -T4 192.168.1.1 -oA full_assessment

# "Find all web servers on a /16"
sudo nmap -sS -T4 -p 80,443,8080,8443,8000,8888,9090,3000,5000 --open 10.0.0.0/16 -oG web_servers.gnmap

# "Find domain controllers"
sudo nmap -sS -p 53,88,135,139,389,445,464,636,3268,3269 --open 192.168.1.0/24

# "Network printer hunt"
sudo nmap -sS -p 9100,515,631 --open 192.168.1.0/24

# "IoT/embedded device discovery"
sudo nmap -sS -p 23,80,443,8080,1883,5683 --open -sV 192.168.1.0/24

# "Find everything with default creds"
sudo nmap -sV --script http-default-accounts,ftp-anon,ms-sql-empty-password,mysql-empty-password -iL live_hosts.txt
```

---

## Operational Guidance & Defensive Considerations

This reference is equally valuable to defenders. Understanding the tactics, techniques, and observed patterns used by reconnaissance operators enables improved detection, hardening, and rule engineering. Defensive teams should consider:

* Explicitly validating source port semantics rather than allowing permissive assumptions.
* Deploying deep-packet inspection for protocols commonly abused for bypass.
* Monitoring for anomalous fragmentation patterns and unusual combinations of flags.
* Ensuring logging and alerting are enabled for scans and malformed packets.
* Detecting decoy scans by correlating IP IDs across suspected decoy sources.
* Implementing stateful firewalls that track connection state (defeats null/FIN/Xmas scans).
* Monitoring for low-and-slow scans by correlating events over longer time windows.
* Rate-limiting connections from single sources to detect and slow enumeration.
* Using TCP RST cookies or SYN cookies to detect half-open scan floods.

---

## Appendix: Raw Notes & Credits

* Excerpts and practical notes inspired from "My Nmap Cheat Sheet" (Aug 7, 2023) by Dan Fedele.
* Reference: https://agrohacksstuff.io/posts/my-nmap-cheat-sheet/
* HackTricks: https://book.hacktricks.wiki/
* Nmap Official Documentation: https://nmap.org/book/man.html
* NSE Script Library: https://nmap.org/nsedoc/
