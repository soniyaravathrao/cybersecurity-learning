# Nmap Notes & Home Lab Practice

My notes from learning Nmap, plus the commands I practiced in my own home lab. All scanning was done against machines I own, inside an isolated virtual network.

## Contents
1. [Overview](#overview)
2. [Home lab setup](#home-lab-setup)
3. [TCP flags](#tcp-flags)
4. [Port states](#port-states)
5. [Choosing targets](#choosing-targets)
6. [Choosing ports](#choosing-ports)
7. [Host discovery](#host-discovery)
8. [Scan types](#scan-types)
9. [OS and service detection](#os-and-service-detection)
10. [Timing templates](#timing-templates)
11. [Firewall and IDS evasion](#firewall-and-ids-evasion)
12. [Output formats](#output-formats)
13. [Commands I practiced in my lab](#commands-i-practiced-in-my-lab)
14. [Legal and ethical use](#legal-and-ethical-use)
15. [My lab results](#my-lab-results)

---

## Overview

**Ethical hacking has 5 phases:**
1. Reconnaissance: gathering information
2. Scanning: mapping hosts, ports and services
3. Exploitation
4. Post-exploitation
5. Reporting

**Nmap (Network Mapper)** is a free, open-source network discovery and security auditing tool. It was created by Gordon Lyon (Fyodor) in 1997 and is still actively maintained. Sysadmins, pentesters and network engineers use it worldwide.

**What Nmap does:**
| Capability | Question it answers |
|---|---|
| Host discovery | Which devices are online? |
| Port scanning | Which ports are open? |
| Service detection | What software is running on them? |
| OS fingerprinting | What operating system is the target? |

**Nmap Scripting Engine (NSE):** scripts written in Lua that automate tasks such as detecting vulnerabilities, finding services, and gathering information from systems.

---

## Home lab setup

- **Attacker:** Kali Linux (VirtualBox)
- **Target:** Metasploitable2, an intentionally vulnerable VM (VirtualBox)
- Keep both VMs on an isolated network (NAT or host-only), never exposed to a real network.

---

## TCP flags

Every scan type is built from TCP flags.

| Flag | Meaning |
|---|---|
| SYN | "I want to connect." |
| ACK | "I received your packets." |
| RST | "Close immediately." |
| FIN | "Done, close gracefully." |
| PSH | "Deliver data now." |
| URG | "Process immediately." |
| NULL | No flags set (used by a stealth scan) |

---

## Port states

How Nmap interprets the response to a probe:

| State | What Nmap sees | Meaning |
|---|---|---|
| **Open** | SYN → SYN+ACK | A service is listening. Example: `22/tcp open ssh` |
| **Closed** | SYN → RST+ACK | Host is up, but nothing is listening. Example: `25/tcp closed smtp` |
| **Filtered** | SYN → no response | A firewall, packet filter or security rule is dropping packets. Example: `443/tcp filtered https` |
| **Unfiltered** | Reachable, but state unclear | Port is reachable, but Nmap can't tell if it is open or closed. Mostly seen in ACK scans. |
| **Open\|filtered / closed\|filtered** | No conclusive response | Ambiguous result |

---

## Choosing targets

| Type | Example | Meaning |
|---|---|---|
| Single IP | `192.168.1.1` | One host |
| Range | `192.168.1.1-254` | 254 hosts |
| CIDR | `192.168.1.0/24` | 256 addresses |
| Hostname | `scanme.nmap.org` | Resolved with DNS |
| File input | `-iL targets.txt` | List of targets from a file |
| Exclude | `--exclude 192.168.1.1` | Skip this host |

**CIDR (Classless Inter-Domain Routing):** the number after `/` says how many bits of the address are fixed.
- `/24` = first 3 numbers fixed, only the last one changes (256 addresses)
- `/16` = first 2 numbers fixed, last two change (65,536 addresses)
- `/8` = first number fixed, the rest change
- Bigger CIDR number = smaller network. Smaller CIDR number = bigger network.

---

## Choosing ports

**Port ranges (IANA):**
| Range | Type | Examples |
|---|---|---|
| 0 - 1023 | Well-known (system services) | HTTP 80, HTTPS 443, SSH 22 |
| 1024 - 49151 | Registered (applications, companies) | MySQL 3306, XMPP 5222 |
| 49152 - 65535 | Dynamic / ephemeral (temporary, client-side) | Assigned by the OS |

*Ephemeral* means short-lived. Example: when you open a website, your browser uses a random port like 52341 to connect to server port 443, and that temporary port disappears when the communication ends.

**Common well-known ports:** 20/21 FTP, 22 SSH, 23 Telnet, 25 SMTP, 53 DNS, 80 HTTP, 110 POP3, 143 IMAP, 443 HTTPS

**Common registered ports:** 1433 Microsoft SQL Server, 1521 Oracle DB, 3306 MySQL, 3389 RDP, 8080 alternate HTTP

**Port syntax:**
| Syntax | Example | Meaning |
|---|---|---|
| Single | `-p 80` | Port 80 only |
| Multiple | `-p 80,443,8080` | Specific ports |
| Range | `-p 1-1024` | Ports 1 to 1024 |
| All | `-p-` | All 65,535 ports |
| Top N | `--top-ports 100` | 100 most common ports |

With no `-p`, Nmap scans the **top 1,000 most common ports**.

```bash
nmap -p 80 <target-ip>
nmap -p 22,80,443 <target-ip>
nmap -p 1-100 <target-ip>
nmap -p- <target-ip>
nmap <target-ip>                      # default: top 1000 ports
nmap -sU -p 53 <target-ip>            # UDP port 53
nmap --exclude-ports 25 <target-ip>
```

---

## Host discovery

Finding live targets before port scanning avoids wasting time on dead hosts.

| Flag | Purpose |
|---|---|
| `-sn` | Discover hosts only, skip the port scan |
| `-Pn` | Skip discovery and assume the host is up (use when ICMP is blocked) |
| `-PS` | TCP SYN ping |
| `-PA` | TCP ACK ping |
| `-PE` | ICMP echo ping |
| `-PP` | ICMP timestamp ping |

**ICMP (Internet Control Message Protocol)** is used for error reporting, network checking and connectivity testing. It helps devices communicate network status information.

---

## Scan types

| Flag | Scan | Needs root? |
|---|---|---|
| `-sS` | TCP SYN (stealth / half-open) | Yes |
| `-sT` | TCP connect (full handshake) | No |
| `-sU` | UDP | Yes |
| `-sN` | NULL | Yes |
| `-sF` | FIN | Yes |
| `-sX` | Xmas | Yes |
| `-sA` | ACK | Yes |
| `-sn` | Ping scan (host discovery only) | No port scan |

### TCP connect scan (`-sT`)
- Completes the full TCP three-way handshake, then closes the connection.
- **Pros:** very reliable, works without root.
- **Cons:** slower and easily logged by the target.
- **Best for:** unprivileged users without raw packet access.

### TCP SYN scan (`-sS`)
- Sends a SYN and never completes the connection. SYN+ACK means open, RST means closed, then Nmap resets the connection.
- **Pros:** faster and less likely to be logged than a full connection.
- **Cons:** needs root/admin (raw socket privileges).
- **Best for:** initial reconnaissance in professional penetration tests.

| | `-sS` | `-sT` |
|---|---|---|
| Root required | Yes | No |
| Full handshake | No | Yes |
| Speed | Faster | Slower |

Both scan **TCP only**.

### UDP scan (`-sU`): the slowest
- UDP has no handshake and no ACK, so there is no clear reply from the target.
- Nmap has to wait for timeouts. Linux systems also rate-limit ICMP replies, so responses are delayed.

| Response | Result |
|---|---|
| UDP reply received | Open |
| ICMP port unreachable (type 3, code 3) | Closed |
| No response after timeout | Open\|filtered (ambiguous) |

Common UDP services: DNS 53, DHCP 67, SNMP 161, NTP 123.

### NULL, FIN and Xmas scans (stealth scans)
A closed port must reply with RST to a packet that doesn't have the expected flags. Nmap uses this behavior: no response means the port is open or filtered.

| Scan | Flags sent | Open response | Closed response |
|---|---|---|---|
| FIN | FIN | No response | RST |
| NULL | None | No response | RST |
| Xmas | FIN + PSH + URG | No response | RST |

These can slip past some simple packet filters, but they don't work reliably against every OS (Windows, for example, replies with RST regardless).

### ACK scan (`-sA`): mapping firewall rules
- Does **not** detect open ports. It shows how a firewall treats packets.
- Sends a bare ACK packet:
  - RST response → **unfiltered** (packet got through)
  - No response → **filtered** (firewall dropped it)
- A **stateful** firewall remembers connections (it knows whether a SYN started a connection). A **stateless** firewall has no memory and treats every packet individually.

---

## OS and service detection

### OS detection (TCP/IP fingerprinting)
Nmap sends specially crafted packets, analyzes how the system responds, and compares the responses with its fingerprint database. It checks details like:
- **TTL (Time To Live):** the number of router hops a packet can make before it is dropped. Different operating systems use different default TTL values.
- **TCP window size:** how the system handles data.
- **IPID pattern:** the ID sequence the OS assigns to packets. Some systems use sequential IDs, others random ones.

```bash
nmap -O <target-ip>     # OS detection only
nmap -A <target-ip>     # aggressive: OS + version + scripts + traceroute
```

### Service version detection (`-sV`)
Nmap connects to open ports, captures service banners, and sends protocol-specific probes that are matched against a large signature database. Exact versions make it possible to look up known CVEs.

Example results from my notes:
- Port 22 → SSH 2.0, OpenSSH 8.2p1 Ubuntu
- Port 80 → Apache 2.4.41

`--version-intensity` ranges from 0 (faster) to 9 (most accurate). The default is 7.

---

## Timing templates

| Flag | Profile | Speed |
|---|---|---|
| `-T0` | Paranoid | Extremely slow (about 1 probe per 5 minutes), maximum stealth |
| `-T1` | Sneaky | Very slow, tries to avoid detection |
| `-T2` | Polite | Slow, low network load |
| `-T3` | Normal | **Default**, balanced |
| `-T4` | Aggressive | Fast, less stealth |
| `-T5` | Insane | Very fast, less accurate, very noisy |

Slower timing makes a scan less likely to trip detection, but it does not make it invisible.

```bash
nmap -sT -T0 <target-ip>
nmap -sT -T4 <target-ip>
nmap -sT -T5 <target-ip>
```

---

## Firewall and IDS evasion

Understanding these techniques matters for defenders too: it shows what scan traffic can look like in logs. Use only in your own lab or with written authorization.

| Technique | Command | Idea |
|---|---|---|
| Fragment packets | `nmap -sS -f <target-ip>` | Splits probes into tiny fragments. Basic filters that inspect only the first fragment can miss the rest. Modern firewalls and IDS usually reassemble fragments. |
| Set packet size | `nmap -sS --mtu 24 <target-ip>` | Sets the fragment size in **bytes** (must be a multiple of 8). |
| Idle / zombie scan | `nmap -sI <zombie-host>:<port> <target-ip>` | Uses a third-party "zombie" host, so the target sees the zombie's IP, not mine. Relies on the zombie's predictable IPID sequence. |
| Source port manipulation | `nmap -sS --source-port 53 <target-ip>` | Some poorly configured firewalls trust traffic coming from port 53 (DNS). |
| Randomize hosts | `nmap --randomize-hosts <network>/24` | Scans targets in random order to break sequential scan detection. |

---

## Output formats

| Flag | Format / purpose |
|---|---|
| `-oN` | Normal text |
| `-oX` | XML |
| `-oG` | Grepable |
| `-oA` | **All three formats at once** (recommended) |
| `-v` | Verbose output (show progress while scanning) |
| `-A` | OS + version + scripts + traceroute |
| `-O` | OS fingerprinting (capital O) |
| `-sV` | Service/version detection |
| `--script` | Run NSE scripts |

Note: `-o` (lowercase) is for output, `-O` (capital) is OS detection.

**Example combined command:**
```bash
nmap -sS -p 1-1000 -T4 -A <target-ip>
```
SYN stealth scan of ports 1-1000, aggressive timing, full enumeration (OS, versions, scripts, traceroute).

---

## Commands I practiced in my lab

**1. Find my own IP and network range (on Kali):**
```bash
ifconfig
```

**2. Discover live hosts on the lab network:**
```bash
nmap <network>/24
# or: nmap 10.x.x.0-255
```
In my VM network, addresses ending in `.1`, `.2`, `.254` and `.255` belonged to the virtual network (gateway, DHCP, broadcast), so I ignored them and looked at the other hosts.

**3. Scan only specific ports instead of all of them:**
```bash
nmap -p 80,443,25,53 <target-ip>
```

**4. Scan all ports, or a range:**
```bash
nmap -p- <target-ip>
nmap -p 1-1024 <target-ip>
```

**5. Full scan saved to a file:**
```bash
sudo nmap -sT -p- -A -v <target-ip> -oX nmapscan.xml
```
- `-p-` all ports, `-A` aggressive (OS + version + scripts + traceroute), `-v` verbose, `-oX` save as XML
- `sudo` is needed because OS detection requires root.

**6. Convert the XML into a readable HTML report:**
```bash
xsltproc nmapscan.xml -o nmapscan.html
```
Then open `nmapscan.html` in a browser to see the summary.

**About `sudo`:** short for "SuperUser Do". It lets a regular user run a command with administrator (root) privileges, for example `sudo apt update`.

---

## Legal and ethical use

**Authorized use:**
- Systems I own, or have written permission to test
- Authorized penetration testing contracts and bug bounty programs (within scope)
- Lab environments

**Safe practice targets:** `scanme.nmap.org`, Hack The Box, VulnHub, my own home lab.

**Rule:** never scan any system without explicit authorization.

---

## My lab results

> Fill this in from my own scan output (`nmapscan.html` or the terminal).

| Port | State | Service | Version |
|---|---|---|---|
| | | | |

**What I found:**
-

**What surprised me / what I'd do differently:**
-

---

## Key takeaways
1. TCP flags (SYN, ACK, RST, FIN, PSH, URG) are the building blocks of every scan.
2. Port states (open, closed, filtered, unfiltered) each tell a different story about the target.
3. Match the scan type to the scenario, from basic `-sT` to stealthier `-sS` and special scans.
4. `-sV`, `-O` and NSE scripts reveal what is running beneath the surface.
5. Use `-oA` to save normal, grepable and XML output at once.
6. Always scan only with authorization.
