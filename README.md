# Operational Reference: Network Reconnaissance & Reconnaissance Evasion

**Classification:** RESTRICTED — FOR AUTHORIZED USE ONLY

**Operational Unit:** Red Cell — Technical Reconnaissance Division

**Author/Source:** Compiled from field notes and operational guidance; includes excerpts from "My Nmap Cheat Sheet" (Posted Aug 7, 2023, by Dan Fedele).



## Purpose

This repository contains a consolidated operational reference for network reconnaissance, scanning techniques, and considerations related to firewall/IDS interaction. It is formatted as a polished, authoritative reference intended for audits, lawful penetration tests, red-team operations, and defensive teams seeking to understand how reconnaissance tools interact with network enforcement points. All material is provided for **authorized, legal, and ethical use only**.

Operators must ensure:

* A signed Rules of Engagement (RoE) or equivalent authorization is in place.
* The scope of activity is explicitly defined and approved.
* All local laws and organizational policies are followed.



## Table of Contents

* Purpose
* Executive Summary
* Best Source Ports for Firewall/IDS Evasion
* Recommended Nmap Command Style ()
* Rationale: Why `--source-port` Can Affect Enforcement
* Ports Not Recommended
* Top Ports for Operational Evasion
* Stealth Combination Example
* Nmap Cheat Sheet (Operational Notes)

  * Typical Usage and Ippsec Special
  * Scan All TCP Ports
  * Scan Specific TCP Ports
  * UDP Scans
  * Host Discovery
  * Useful Flags and Timing
  * Logging and Output
  * Proxy and Script Usage
* Nmap Scripts:  and Guidance
* Appendix: Raw Notes & Credits



## Summary

Networks and enforcement appliances are occasionally misconfigured or rely on legacy assumptions about service behavior. In operational assessments, understanding which source ports and scan techniques are more likely to pass through permissive or misconfigured enforcement is useful for both offensive and defensive teams. This document compiles commonly observed source ports, command patterns, and a concise nmap reference.



## Best Source Ports for Firewall/IDS Evasion

These source ports are commonly whitelisted, treated as trusted, or otherwise subject to less strict inspection due to historical service expectations or misconfiguration:

1. **53 (DNS)** — Frequently allowed for both inbound and outbound DNS traffic.
2. **20 / 21 (FTP Active / Control)** — Some stateless or legacy firewalls treat FTP-related traffic as trusted.
3. **80 (HTTP)** — Often allowed where web traffic is permitted; source port 80 can be less inspected.
4. **443 (HTTPS)** — Similar to port 80 and, in many deployments, subjected to even less inspection.
5. **123 (NTP)** — Some networks permit NTP broadly; IDS may not thoroughly inspect.
6. **179 (BGP)** — Older routing equipment may implicitly trust BGP ports in specific configurations.
7. **500 / 4500 (IPsec)** — Corporate firewalls that support IPsec may whitelist these ports.
8. **67 / 68 (DHCP)** — Often trusted or ignored on internal networks.

The presence of these ports on the list reflects operational observation of misconfigurations and permissive policies; it is not a guarantee that they will succeed in any environment.



## Recommended Nmap Command Style 

```
sudo nmap -Pn --source-port 53 -f 10.20.30.40
```

```
sudo nmap -Pn -f --mtu 16 --source-port 53 -D RND:5 10.20.30.40
```

These  combine source-port selection and packet fragmentation and are included as case studies of common command constructions found in operational playbooks.



## Rationale: Why `--source-port` Can Affect Enforcement

Some enforcement systems make legacy assumptions about traffic semantics (for instance: "traffic from port 53 must be DNS"), which can lead to overly permissive rules. Using a particular source port may cause a firewall or ACL to treat probe traffic differently, enabling reconnaissance in environments with such misconfigurations.

Such techniques can interact with:

* Stateless packet filters
* Misconfigured cloud security groups
* ACL-based vendor appliances (legacy Cisco, MikroTik, SonicWall, etc.)



## Ports NOT Recommended

Avoid using the following unless operationally required and authorized:

* **0** — Often dropped or explicitly logged by enforcement devices.
* **High ephemeral ranges (49152+)** — Typical of normal outbound traffic; unlikely to evade inspection.
* **Well-monitored service ports** (SSH, SMTP, POP, IMAP) — Frequently subject to logging and active monitoring.



## Top 3 for Operational Evasion 

1. Port 53
2. Port 80
3. Port 443

These ports are most commonly observed to pass through permissive enforcement controls in diverse environments, but success is environment dependent.



## Stealth Combination 

pattern combining fragmentation, trusted source port, and decoys:

```
sudo nmap -Pn -f --mtu 16 --source-port 53 -D RND:5 10.20.30.40
```

This pattern is included as an operational note and should only be used within an authorized engagement.



## Nmap Cheat Sheet — Operational Notes


Nmap is a fundamental reconnaissance tool for network assessments. The notes below collect typical command patterns, flags, and considerations used by experienced operators.

### Typical Usage (Ippsec Special)

```
sudo nmap -sC -sV -oN portscan -v 10.20.30.40
```

* `-sC` runs default scripts
* `-sV` attempts service/version detection
* `-oN` saves normal output to a file
* `-v` increases verbosity

Running as root (via `sudo`) enables SYN scans and other low-level techniques that are restricted to privileged users.

### Scan All TCP Ports

```
sudo nmap -p- -oN allports -v 10.20.30.40
```

### Scan Specific TCP Ports

```
sudo nmap -sC -sV -p 22,25,80,443,5900-5999 -oN portscan 10.20.30.40
```

### UDP Scans

```
sudo nmap -sU -oN udp-portscan -v 10.20.30.40
```

Note: UDP scans can be slow and produce noisy results due to the nature of the protocol.

### Scan ALL UDP Ports

```
sudo nmap -sU -p- -oN udp-im-running-out-of-options -v 10.20.30.40
```

### Bypass Some IDS and Firewalls (Illustrative)

```
sudo nmap -Pn --source-port=9090 -f 10.20.30.40
```

YMMV. This is an operational pattern observed in the field where source-port selection and packet fragmentation were used together.

### RING ALL THE BELLS

```
sudo nmap -p- -T5 --max-retries 0 -v -oA allports --script /usr/share/nmap/scripts/ 10.20.30.40
```

This will scan the target with “extreme aggression.” Also sets retransmissions to no cap so it will keep banging on the door until someone answers, basically. When they do answer, throw the kitchen sink at them by attempting to run all available scripts against the port, throwing caution to the wind. This is what you run when you want them to know you’re there.


### Host Discovery: Ping-sweep an Entire Subnet

```
sudo nmap -sn 10.20.30.0/24
```

ICMP may be blocked. Nmap uses several techniques; behavior depends on privileges and network topology.

For VMWare NAT environments where `-sn` yields false positives, `--unprivileged` may be used as a mitigation when running without low-level packet capture.

```
sudo nmap -sn --unprivileged 10.20.30.0/24
```
It’s been brought to my attention that performing a ping sweep in a very specific set of conditions, namely running a virtual machine inside VMWare with your network settings configured to NAT (which is the default), that you may get incorrect output in that all hosts in the subnet will be considered up. The reason for this is because when operating with the -sn flag, nmap will check a target host is up using the following methods:

```
nmap receives ICMP reply to ICMP ECHO_REQUEST packet
nmap receives ICMP reply to ICMP TIMESTAMP_REQUEST packet
nmap receives TCP SYN/ACK reply to 443/TCP SYN packet
nmap receives TCP RST reply to 80/TCP ACK packet
```


### Host Discovery: List Scan

```
sudo nmap -sL 10.20.30.0/24
```

This performs DNS reverse lookups and is covert but does not actively probe hosts.



### Useful Flags

* **OS Detection:** `-O`
* **Timing Template:** `-T 0..5` (0 = paranoid, 5 = aggressive)
* **Rate Control:** `--min-rate` / `--max-rate`
* **Skip Host Discovery:** `-Pn`
* **No DNS Resolution:** `-n`
* **Output formats:** `-oN`, `-oA`, `-oG`, `-oX`



### Nmap Through a Proxy

```
sudo nmap -sT --proxy socks4://40.40.50.50:5789 10.20.30.40
```

Or via proxychains:

```
sudo proxychains nmap 10.20.30.40
```



## Nmap Scripts:  and Guidance

Nmap scripting engine (NSE) provides numerous scripts for enumeration and vulnerability checks. Use scripts selectively and with awareness of potential impact.



```
sudo nmap -p 22,80,443 --script "vuln and safe" 10.20.30.40
```

```
sudo nmap -p 80 --script http-enum 10.20.30.40
```

```
sudo nmap -p 139,445 --script smb-enum-users --script-args 'smbusername="admin",smbpassword="password"' 10.20.30.40
```

## LDAP 

```
sudo nmap -p 389 --script ldap-rootdse 10.20.30.40
```

```
sudo nmap -p 389 --script ldap-search --script-args 'ldap.username="",ldap.password=""' 10.20.30.40
```

Script-based enumeration can be useful but other specialized tools may be more flexible for some tasks (e.g., ldapsearch, CrackMapExec).


## Operational Guidance & Defensive Considerations

This reference is equally valuable to defenders. Understanding the tactics, techniques, and observed patterns used by reconnaissance operators enables improved detection, hardening, and rule engineering. Defensive teams should consider:

* Explicitly validating source port semantics rather than allowing permissive assumptions.
* Deploying deep-packet inspection for protocols commonly abused for bypass.
* Monitoring for anomalous fragmentation patterns and unusual combinations of flags.
* Ensuring logging and alerting are enabled for scans and malformed packets.



## Appendix: Raw Notes & Credits

* Excerpts and practical notes inpired from "My Nmap Cheat Sheet" (Aug 7, 2023) by Dan Fedele.
* Refrence https://agrohacksstuff.io/posts/my-nmap-cheat-sheet/

