# Network Enumeration with Nmap — HTB Academy Lab Write-up

**Author:** Nischal Dhakal  
**Platform:** Hack The Box Academy  
**Module:** Network Enumeration with Nmap

> **Important:** HTB lab IP addresses change between sessions. Replace every example IP address in this write-up with the address assigned to your own lab instance.

## Overview

This write-up documents the commands and reasoning I used to complete the practical questions in the HTB Academy **Network Enumeration with Nmap** module.

The focus is not only on reaching the answer, but also on understanding what each scan reveals and why a particular Nmap option is useful.

---

## 1. Host Discovery

### Question

> Based on the last result, determine which operating system the target belongs to and submit the operating system name.

### Command and packet trace

```bash
sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --disable-arp-ping
```

Example output:

```text
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 00:12 CEST
SENT (0.0107s) ICMP [10.10.14.2 > 10.129.2.18 Echo request (type=8/code=0) id=13607 seq=0] IP [ttl=255 id=23541 iplen=28 ]
RCVD (0.0152s) ICMP [10.129.2.18 > 10.10.14.2 Echo reply (type=0/code=0) id=13607 seq=0] IP [ttl=128 id=40622 iplen=28 ]
Nmap scan report for 10.129.2.18
Host is up (0.086s latency).
MAC Address: DE:AD:00:00:BE:EF
Nmap done: 1 IP address (1 host up) scanned in 0.11 seconds
```

### Understanding the command

- `-sn` performs host discovery without running a port scan.
- `-oA host` saves the result in Nmap's normal, grepable, and XML formats.
- `-PE` sends an ICMP Echo Request.
- `--packet-trace` displays the packets Nmap sends and receives.
- `--disable-arp-ping` prevents Nmap from using ARP discovery.

### How the answer was identified

The important part of the response is:

```text
Echo reply ... ttl=128
```

TTL stands for **Time to Live**. It prevents packets from circulating indefinitely. Each router that forwards a packet reduces its TTL value by one. When the value reaches zero, the packet is discarded.

The TTL in a reply can provide a clue about the operating system because different systems commonly use different initial TTL values.

- Windows systems commonly begin with a TTL of `128`.
- Linux and Unix-like systems commonly begin with a TTL of `64`.
- Some network devices use a higher initial TTL.

Because the received Echo Reply had a TTL of `128`, the result indicated a **Windows** system.

### Answer

```text
Windows
```

---

## 2. Host and Port Scanning

### Question 1

> Find all TCP ports on the target and submit the total number of discovered TCP ports.

### Command

```bash
sudo nmap -p- 10.129.121.198
```

### Explanation

The `-p-` option tells Nmap to scan all TCP ports from `1` through `65535`.

After the scan completes, count the ports reported as open. The number of open TCP ports is the required answer.

> The exact result depends on the target assigned to the current lab session.

---

### Question 2

> Enumerate the hostname of the target and submit it exactly as displayed. The answer is case-sensitive.

### Command

```bash
sudo nmap -sC 10.129.121.198
```

### Explanation

The `-sC` option runs Nmap's default NSE script set against the discovered services.

In this lab, the hostname can be found in the **Host script results**, specifically in the output from the `smb-os-discovery` script.

Look for a line similar to:

```text
NetBIOS computer name: <HOSTNAME>
```

Copy the hostname exactly because the answer is case-sensitive.

---

## 3. Saving Nmap Results

### Question

> Perform a full TCP port scan, create an HTML report, and submit the highest discovered port number.

### Nmap output formats

Nmap can save scan results in several formats:

- `-oN` — normal output, commonly saved with a `.nmap` extension
- `-oG` — grepable output, commonly saved with a `.gnmap` extension
- `-oX` — XML output, saved with a `.xml` extension
- `-oA` — saves normal, grepable, and XML output at the same time

### Generate XML output

```bash
sudo nmap -p- 10.129.121.198 -oX report.xml
```

The scan checks all TCP ports and saves the result as XML.

Review the discovered ports and submit the highest open port number shown in the results.

### Convert XML to HTML

```bash
xsltproc style.xsl report.xml > output.html
```

The resulting `output.html` file can be opened in a browser.

> The stylesheet path may differ depending on the system and how Nmap is installed.

---

## 4. Service Enumeration

### Question

> Enumerate all open ports and their services. One of the services contains the flag that must be submitted.

### Command

```bash
sudo nmap -sV -p22,80,110,139,143,445,31337 10.129.121.198
```

### Explanation

The `-sV` option enables service and version detection.

Instead of only reporting that a port is open, Nmap probes the service and attempts to identify what is running on it.

The ports discovered during the earlier scan were supplied explicitly:

```text
22, 80, 110, 139, 143, 445, 31337
```

Review the service banners and version information carefully. One of the services exposes the required flag in its scan output.

---

## 5. Nmap Scripting Engine

### Question

> Use NSE scripts to locate the flag exposed by one of the services.

### Command

```bash
sudo nmap 10.129.121.198 -p 80 -sV --script vuln
```

### Explanation

- `-p 80` scans only the web service.
- `-sV` attempts to identify the service and version.
- `--script vuln` runs NSE scripts from the vulnerability category.

The scan revealed a `robots.txt` file. In CTF-style environments, `robots.txt` is worth checking because it may disclose hidden or restricted web paths.

### Retrieve `robots.txt`

```bash
curl http://10.129.121.198/robots.txt
```

I then visited the path exposed by `robots.txt` and obtained the required flag.

---

## 6. Firewall and IDS/IPS Evasion — Easy Lab

### Question

> Identify the operating system running on the client's machine and submit the operating system name.

### Command

```bash
sudo nmap -O -sV 10.129.121.206
```

### Explanation

- `-O` enables operating system detection.
- `-sV` enables service and version detection.

The answer required the specific operating system distribution, not only the general operating system family.

For example, `Linux` is the family, while `Ubuntu` is the specific distribution identified in the scan.

### Answer

```text
Ubuntu
```

---

## 7. Firewall and IDS/IPS Evasion — Medium Lab

### Question

> Determine the version of the target's DNS server and submit it.

### Command

```bash
sudo nmap -sU -p53 --script dns-nsid 10.129.2.48
```

### Explanation

- `-sU` performs a UDP scan.
- `-p53` scans only DNS on UDP port 53.
- `--script dns-nsid` runs an NSE script that asks the DNS server for identifying information.

Review the script output for the DNS implementation and version returned by the server. Submit the version exactly as shown in your own lab result.

---

## 8. Firewall and IDS/IPS Evasion — Hard Lab

### Initial scan

```bash
sudo nmap -n -Pn -sS -p- --source-port 53 --max-rate 100 --max-retries 2 10.129.121.211
```

### Explanation

- `-n` disables DNS resolution.
- `-Pn` skips host discovery and treats the target as online.
- `-sS` performs a TCP SYN scan.
- `-p-` scans all TCP ports.
- `--source-port 53` sends probes using local source port 53.
- `--max-rate 100` limits the scan to a maximum of 100 packets per second.
- `--max-retries 2` limits how often Nmap retransmits unanswered probes.

The important option in this lab was:

```text
--source-port 53
```

Some poorly configured firewalls trust traffic that appears to originate from port 53 because DNS responses commonly use that source port. Using source port 53 allowed the scan to identify an otherwise filtered service.

The scan showed that TCP port `50000` was open.

### Connect to the discovered service

```bash
nc -nv -p 53 10.129.107.56 50000
```

### Explanation

- `-n` disables name resolution.
- `-v` enables verbose output.
- `-p 53` uses local source port 53.
- `50000` is the destination port discovered during the Nmap scan.

After connecting, I listed the available files and found the flag in the working directory.

> The IP address used for the Netcat connection must match the active target in your own lab session.

---

## Key Lessons

This lab reinforced several important Nmap concepts:

1. Packet traces and TTL values can provide clues about the target operating system.
2. `-p-` is necessary when a service may be running outside the default top 1,000 ports.
3. NSE scripts can reveal hostnames, web paths, service information, and other useful details.
4. XML scan output can be transformed into an HTML report.
5. Service and version detection helps turn open ports into actionable information.
6. UDP enumeration is essential when investigating services such as DNS.
7. Firewall behavior can sometimes be tested by adjusting packet properties such as the source port.
8. Scan results must always be interpreted in the context of the active lab target.

---

## Command Summary

```bash
# Host discovery with packet tracing
sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --disable-arp-ping

# Full TCP port scan
sudo nmap -p- 10.129.121.198

# Default NSE scripts
sudo nmap -sC 10.129.121.198

# XML report
sudo nmap -p- 10.129.121.198 -oX report.xml

# Convert XML to HTML
xsltproc style.xsl report.xml > output.html

# Service/version detection
sudo nmap -sV -p22,80,110,139,143,445,31337 10.129.121.198

# Vulnerability-category NSE scripts
sudo nmap 10.129.121.198 -p 80 -sV --script vuln

# Read robots.txt
curl http://10.129.121.198/robots.txt

# OS and service detection
sudo nmap -O -sV 10.129.121.206

# DNS version enumeration
sudo nmap -sU -p53 --script dns-nsid 10.129.2.48

# Hard-lab scan using source port 53
sudo nmap -n -Pn -sS -p- --source-port 53 --max-rate 100 --max-retries 2 10.129.121.211

# Connect to the discovered high port using source port 53
nc -nv -p 53 10.129.107.56 50000
```

---

## Disclaimer

These techniques were used in an authorized Hack The Box Academy lab. Only scan systems you own or have explicit permission to test.
