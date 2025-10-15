## Scanning Options

| **Nmap Option** | **Description** |
|---|----|
| `10.10.10.0/24` | Target network range. |
| `-sn` | Disables port scanning. |
| `-Pn` | Disables ICMP Echo Requests |
| `-n` | Disables DNS Resolution. |
| `-PE` | Performs the ping scan by using ICMP Echo Requests against the target. |
| `--packet-trace` | Shows all packets sent and received. |
| `--reason` | Displays the reason for a specific result. |
| `--disable-arp-ping` | Disables ARP Ping Requests. |
| `--top-ports=<num>` | Scans the specified top ports that have been defined as most frequent.  |
| `-p-` | Scan all ports. |
| `-p22-110` | Scan all ports between 22 and 110. |
| `-p22,25` | Scans only the specified ports 22 and 25. |
| `-F` | Scans top 100 ports. |
| `-sS` | Performs an TCP SYN-Scan. |
| `-sA` | Performs an TCP ACK-Scan. |
| `-sU` | Performs an UDP Scan. |
| `-sV` | Scans the discovered services for their versions. |
| `-sC` | Perform a Script Scan with scripts that are categorized as "default". |
| `--script <script>` | Performs a Script Scan by using the specified scripts. |
| `-O` | Performs an OS Detection Scan to determine the OS of the target. |
| `-A` | Performs OS Detection, Service Detection, and traceroute scans. |
| `-D RND:5` | Sets the number of random Decoys that will be used to scan the target. |
| `-e` | Specifies the network interface that is used for the scan. |
| `-S 10.10.10.200` | Specifies the source IP address for the scan. |
| `-g` | Specifies the source port for the scan. |
| `--dns-server <ns>` | DNS resolution is performed by using a specified name server. |




## Output Options


| **Nmap Option** | **Description** |
|---|----|
| `-oA filename` | Stores the results in all available formats starting with the name of "filename". |
| `-oN filename` | Stores the results in normal format with the name "filename". |
| `-oG filename` | Stores the results in "grepable" format with the name of "filename". |
| `-oX filename` | Stores the results in XML format with the name of "filename". |



## Performance Options

| **Nmap Option** | **Description** |
|---|----|
| `--max-retries <num>` | Sets the number of retries for scans of specific ports. |
| `--stats-every=5s` | Displays scan's status every 5 seconds. |
| `-v/-vv` | Displays verbose output during the scan. |
| `--initial-rtt-timeout 50ms` | Sets the specified time value as initial RTT timeout. |
| `--max-rtt-timeout 100ms` | Sets the specified time value as maximum RTT timeout. |
| `--min-rate 300` | Sets the number of packets that will be sent simultaneously. |
| `-T <0-5>` | Specifies the specific timing template. |



## NSE SCRIPTS 
Now let us move on to HTTP port 80 and see what information and vulnerabilities we can find using the vuln category from NSE.
https://nmap.org/nsedoc/scripts/
```c
 sudo nmap 10.129.2.28 -p 80 -sV --script vuln
```

# Nmap — Firewall & IDS/IPS Evasion (cheat-sheet)

A short, copy-pasteable **.md** summary for quick reference — put this in your Nmap quick sheet.

---

## Concepts

* **Firewall** — inspects/filters traffic; can *drop* packets (no reply) or *reject* (explicit RST for TCP, ICMP error codes).
* **IDS/IPS** — passive (IDS) or active (IPS) monitoring using signatures/patterns. IPS can block attacker IPs. Detection is harder than firewall evasion.
* **Goal** — determine filtering rules and evade/avoid detection by changing packet characteristics, origin, or timing.

---

## Useful scan types & what they reveal

* **SYN scan** `-sS`
  Standard stealth scan. If host replies `SYN+ACK` → likely open. If ICMP unreachable or no reply → filtered.
* **ACK scan** `-sA`
  Sends only ACK flag. Firewalls often allow ACK through; results show *filtered* vs *unfiltered* (not open/closed). Useful to map firewall rules.
* **OS detection** `-O`
  Fingerprint OS. Can be impacted by firewall/filtering — may need open/closed ports visible to be reliable.

---

## Evasion techniques & flags

* **Disable host discovery / DNS / ARP**

  * `-Pn` disable ICMP/ping host discovery
  * `-n` disable DNS resolution
  * `--disable-arp-ping` (when on same LAN and you want to avoid ARP)
* **Packet tracing**

  * `--packet-trace` show sent/received packets (helpful for debugging behavior)
* **Change source port**

  * `--source-port <port>` (or short `--source-port 53`) — use trusted port (e.g., 53) to pass firewall rules that allow DNS.
* **Change source IP (spoof / alternate)**

  * `-S <ip>` specify source IP (requires privileges and network support)
  * `-e <iface>` specify interface to send from (use with `-S` if routing/tunneling)
* **Decoys**

  * `-D <decoy1,decoy2,...,ME,...>` or `-D RND:<n>` random decoys. Inserts fake source IPs among which your real IP is placed. Useful to obscure origin but decoys **must be alive** or SYN-flood protections may trigger.
* **Use different DNS servers**

  * `--dns-server <ns1,ns2>` — run scans via DNS servers that the target trusts (useful in DMZ scenarios).
* **IP ID / header tweaks**

  * Nmap has options to tweak IP/TCP header fields (advanced usage) to evade simple signature matching.

---

## Practical strategies & notes

* **Detecting firewall behavior:**

  * If ports are `filtered` → packets likely dropped (no reply) or ICMP unreachable returned.
  * If `rejected` → RST or ICMP error returned (explicit).
* **ACK vs SYN comparison** exposes firewall behavior:

  * Example: SYN (`-sS`) might get `SYN-ACK` on port 22 (open) and ICMP for others → shows some ports filtered.
  * ACK (`-sA`) often returns RST for open/closed, can reveal which ports are filtered by firewall rules.
* **Detecting IDS/IPS:**

  * Use multiple VPS IPs; if one VPS gets blocked mid-test, IPS likely present.
  * If a single aggressive scan triggers blocks, reduce scan noisiness and use decoys / rate-limiting.
* **When a filtered port opens by changing source port/IP:**

  * If `--source-port 53` makes a previously filtered port show `open`, firewall trusts DNS source port — exploit to route traffic or probe further.
* **Spoofing caveats:**

  * Many ISPs/routers drop spoofed packets. Use real VPSs as decoys or `-S` from allowed subnet when possible.
* **Decoy caveats:**

  * If decoys are offline, target may detect anomaly (SYN-flood protections), causing access issues.

---

## Common useful command patterns (examples)

```bash
# SYN scan, limited ports, packet trace
sudo nmap 10.129.2.28 -p21,22,25 -sS -Pn -n --disable-arp-ping --packet-trace

# ACK scan
sudo nmap 10.129.2.28 -p21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace

# SYN scan using decoys (5 random decoys)
sudo nmap 10.129.2.28 -p80 -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5

# Use different source IP and interface (requires routing/tunnel)
sudo nmap 10.129.2.28 -n -Pn -p445 -O -S 10.129.2.200 -e tun0

# SYN scan using source port 53 (bypass firewall allowing DNS)
sudo nmap 10.129.2.28 -p50000 -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53

# Quick OS detection (may show filtered if firewall blocks probes)
sudo nmap 10.129.2.28 -n -Pn -p445 -O
```

## Post-scan test (connect when port allowed via source port trick)

```bash
# Connect with ncat using source port 53
ncat -nv --source-port 53 10.129.2.28 50000
# → if connected, you may see service banner (e.g., "220 ProFTPd")
```

---

## Quick reference table

| Goal                       |                     Flag(s) | Notes                                        |
| -------------------------- | --------------------------: | -------------------------------------------- |
| SYN scan                   |                       `-sS` | Standard stealth scan                        |
| ACK scan                   |                       `-sA` | Map firewall rules (filtered vs unfiltered)  |
| Decoys                     | `-D RND:5` or `-D a,b,ME,c` | Obfuscate source IP; decoys must be alive    |
| Use trusted source port    |          `--source-port 53` | Bypass firewalls trusting DNS                |
| Spoof source IP            |                   `-S <ip>` | Requires routing privileges; ISP might block |
| OS detection               |                        `-O` | Needs open/closed ports for accuracy         |
| Disable DNS/host discovery |                    `-n -Pn` | Quieter scans                                |
| Interface                  |                `-e <iface>` | Use tunnel interface (e.g., tun0)            |
| Packet-level debug         |            `--packet-trace` | See actual packets sent/received             |

---

## Key takeaways

* Compare **SYN** vs **ACK** scans to identify firewall behavior.
* Use **source port** and **decoys** to bypass naive firewall rules and hide origin.
* Use **multiple VPSes** to test for IPS (if one IP gets blocked, IPS likely present).
* Spoofing is limited by ISP/router filtering — prefer real alternative hosts when possible.
* Always be mindful of legality and authorization — only test networks you are permitted to.

---

