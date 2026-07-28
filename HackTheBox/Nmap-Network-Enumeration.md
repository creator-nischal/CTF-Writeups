# Nmap Network Enumeration Notes

**Platform:** Hack The Box  
**Topic:** Network discovery, service enumeration, scan output, and firewall testing  
**Author:** Nischal Dhakal

> **Legal and ethical notice:** Run these commands only against systems you own or are explicitly authorized to test, such as assigned Hack The Box targets.

---

## Overview

Nmap is a network discovery and enumeration tool. During a penetration test or lab, it helps identify:

- reachable hosts
- open, closed, and filtered ports
- services listening on those ports
- service versions
- possible operating-system details
- selected misconfigurations and known vulnerability indicators

A common starting scan is:

```bash
nmap -sC -sV <target-ip>
```

For example:

```bash
nmap -sC -sV 10.129.2.28
```

### Options used

- `-sC` runs Nmap's default NSE scripts.
- `-sV` attempts to identify service versions.

This scan provides a useful first view of the target, but it does not scan every TCP port unless `-p-` is added.

---

## Tracing Nmap Packets

Packet tracing helps show how Nmap communicates with a target and how the target responds.

```bash
sudo nmap 10.129.2.28 -p 21 --packet-trace -Pn -n --disable-arp-ping
```

### Options used

- `-p 21` scans TCP port 21 only.
- `--packet-trace` displays packets sent and received by Nmap.
- `-Pn` skips normal host discovery and treats the target as online.
- `-n` disables reverse-DNS resolution.
- `--disable-arp-ping` disables ARP-based host discovery on a local Ethernet network.

Packet tracing is especially useful when learning the difference between scan types or investigating why a port appears filtered.

> The original notes referenced a local screenshot here. The image was not included with the uploaded Markdown file, so it has not been added to this version.

---

## Scanning All TCP Ports

By default, Nmap scans its 1,000 most common TCP ports. To scan all 65,535 TCP ports, use `-p-`:

```bash
sudo nmap 10.129.2.28 -p-
```

A full-port scan can reveal services running on uncommon ports that a default scan would miss.

A practical workflow is:

1. Run a full TCP-port scan.
2. Note the open ports.
3. Run focused service and script scans against those ports.

For example:

```bash
sudo nmap 10.129.2.28 -p- -oA target
```

---

## Saving Nmap Results

Saving scan output is important for evidence, later analysis, and reporting.

### Normal output

```bash
sudo nmap 10.129.2.28 -p- -oN target.nmap
```

`-oN` saves human-readable normal output.

### Grepable output

```bash
sudo nmap 10.129.2.28 -p- -oG target.gnmap
```

`-oG` saves grep-friendly output. This format is considered legacy, but it can still be useful for simple command-line processing.

### XML output

```bash
sudo nmap 10.129.2.28 -p- -oX target.xml
```

`-oX` saves structured XML output. XML is useful for automation, parsing, and conversion into reports.

### Save all major formats

```bash
sudo nmap 10.129.2.28 -p- -oA target
```

`-oA target` creates:

- `target.nmap`
- `target.gnmap`
- `target.xml`

The XML output can also be transformed into HTML for easier presentation to non-technical readers.

---

## Focused Service Enumeration

After identifying open ports, scan only those ports with version detection and suitable NSE scripts.

For example, if ports 22, 80, and 445 are open:

```bash
sudo nmap 10.129.2.28 -p 22,80,445 -sC -sV
```

This is normally more efficient than repeatedly running expensive scripts against every port.

---

## Vulnerability Script Scanning

Nmap includes NSE scripts grouped under the `vuln` category.

```bash
sudo nmap 10.129.2.28 -p 80 -sV --script vuln
```

This command:

- checks port 80
- performs service-version detection
- runs scripts classified in the `vuln` category

### Important limitation

The result should be treated as a lead, not final proof of a vulnerability. Script output may include false positives, incomplete checks, or findings based only on a detected version. Confirm important results manually before reporting them.

---

## SYN Scan

A SYN scan is commonly used for TCP-port discovery:

```bash
sudo nmap 10.129.2.28 -p 21,22,25 -sS -Pn -n --disable-arp-ping --packet-trace
```

### How it works

Nmap sends a TCP SYN packet:

- `SYN/ACK` usually indicates an open port.
- `RST` usually indicates a closed port.
- no response or certain ICMP errors may indicate filtering.

Because the full TCP connection is normally not completed, this is often called a half-open scan.

---

## ACK Scan and Firewall Testing

An ACK scan is mainly used to study firewall filtering behavior:

```bash
sudo nmap 10.129.2.28 -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace
```

### What an ACK scan tells us

An ACK scan does **not** normally determine whether a TCP port is open.

Its main results are:

- `unfiltered`: the probe reached the host and produced a TCP reset
- `filtered`: a firewall or packet filter appears to have blocked the probe

This makes `-sA` useful for mapping firewall rules and comparing them with the results of a SYN scan.

### SYN scan versus ACK scan

| Scan | Main purpose | Typical result states |
|---|---|---|
| `-sS` | Find open TCP ports | open, closed, filtered |
| `-sA` | Test filtering behavior | unfiltered, filtered |

> The original notes referenced another local screenshot in this section. It was not included in the uploaded file.

---

## Decoy Scanning

Nmap can place decoy addresses among scan probes:

```bash
sudo nmap 10.129.2.28 -p 80 -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5
```

`-D RND:5` adds five randomly generated decoy addresses.

### What decoys do

Decoys may make logs show multiple apparent scanning sources, making the true scanner less obvious during basic log review.

### Limitations

Decoys are not a dependable method for bypassing:

- geographic IP blocking
- access-control lists that block the real source
- modern IDS/IPS correlation
- routing restrictions

Your real source address still participates in the scan. Decoys should therefore be understood as an obfuscation technique, not as a guaranteed firewall bypass.

---

## Source Address Spoofing

The original notes included the following command:

```bash
sudo nmap 10.129.2.28 -n -Pn -p 445 -O -S 10.129.2.200 -e tun0
```

### Options used

- `-S 10.129.2.200` sets the claimed source IP address.
- `-e tun0` selects the network interface.
- `-O` enables operating-system detection.

### Important limitation

When a source IP is spoofed, replies are normally sent to the spoofed address rather than to the scanning system. This means a spoofed scan often cannot receive the responses needed to determine accurate port states or operating-system details.

Source spoofing only works meaningfully in specialized network conditions where the tester can observe or control the return traffic. It should not be treated as a normal replacement for scanning from the actual source IP.

---

## Operating-System Detection

Use `-O` to attempt operating-system fingerprinting:

```bash
sudo nmap -O 10.129.2.28
```

OS detection compares network responses with Nmap's fingerprint database.

The result is an estimate. Accuracy is affected by:

- firewalls
- filtered ports
- virtualized systems
- insufficient open and closed ports
- network devices modifying packets

For better results, combine OS detection with service enumeration:

```bash
sudo nmap 10.129.2.28 -O -sV
```

---

## DNS Service Enumeration

DNS usually uses UDP port 53, although TCP port 53 is also used in some situations.

The original command was:

```bash
sudo nmap -sU -p 53 --script dns-nsid 10.129.2.48
```

### Options used

- `-sU` performs a UDP scan.
- `-p 53` targets DNS.
- `--script dns-nsid` queries supported DNS server identity information.

Depending on the server, the script may reveal details such as an NSID, hostname, or version-related identifier. A server may also return no useful information.

A broader DNS service scan could be:

```bash
sudo nmap 10.129.2.48 -sU -sV -p 53 --script dns-nsid
```

---

## Reducing Noise During Scanning

The most reliable way to reduce unnecessary traffic is to scan only what is needed.

Instead of repeatedly scanning every port with every script:

1. discover open ports
2. identify the relevant services
3. run focused scripts against those ports

For example:

```bash
sudo nmap 10.129.2.28 -p 80,443 -sC -sV
```

This approach is:

- faster
- easier to review
- less noisy
- less likely to trigger unnecessary alerts
- safer for fragile services

Reducing scan scope does not guarantee firewall or IDS/IPS evasion. It simply produces less traffic and keeps the assessment focused.

---

## Practical Enumeration Workflow

### 1. Initial service scan

```bash
nmap -sC -sV 10.129.2.28
```

### 2. Full TCP-port scan

```bash
sudo nmap 10.129.2.28 -p- -oA target-full
```

### 3. Focused scan of discovered ports

```bash
sudo nmap 10.129.2.28 -p <open-ports> -sC -sV -oA target-services
```

Example:

```bash
sudo nmap 10.129.2.28 -p 22,80,445 -sC -sV -oA target-services
```

### 4. Run protocol-specific scripts

```bash
sudo nmap 10.129.2.48 -sU -p 53 -sV --script dns-nsid
```

### 5. Validate important findings manually

Nmap helps identify attack surface, but it does not replace manual enumeration. Every meaningful finding should be reviewed using protocol-specific tools and direct interaction with the service.

---

## Key Takeaways

- `-sC -sV` is a strong initial service-enumeration combination.
- `-p-` scans all TCP ports.
- `-oA` saves normal, grepable, and XML output together.
- `--packet-trace` is useful for understanding Nmap's behavior.
- `-sS` discovers TCP-port states, while `-sA` mainly tests firewall filtering.
- `--script vuln` can identify leads, but findings require validation.
- Decoys provide limited source obfuscation and are not a guaranteed bypass.
- Spoofed-source scans usually cannot receive the target's replies.
- Focused scanning is more efficient and produces cleaner evidence.

---

## Command Reference

```bash
# Default scripts and service detection
nmap -sC -sV 10.129.2.28

# Trace packets to a specific port
sudo nmap 10.129.2.28 -p 21 --packet-trace -Pn -n --disable-arp-ping

# Scan all TCP ports and save all main output formats
sudo nmap 10.129.2.28 -p- -oA target

# Run vulnerability-category scripts on HTTP
sudo nmap 10.129.2.28 -p 80 -sV --script vuln

# ACK scan for firewall analysis
sudo nmap 10.129.2.28 -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace

# SYN scan for TCP-port discovery
sudo nmap 10.129.2.28 -p 21,22,25 -sS -Pn -n --disable-arp-ping --packet-trace

# Decoy scan
sudo nmap 10.129.2.28 -p 80 -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5

# Attempt operating-system detection
sudo nmap -O 10.129.2.28

# Query DNS NSID information over UDP
sudo nmap -sU -p 53 --script dns-nsid 10.129.2.48
```
