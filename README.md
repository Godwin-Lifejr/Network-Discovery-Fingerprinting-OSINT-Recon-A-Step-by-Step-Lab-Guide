# Network-Discovery-Fingerprinting-OSINT-Recon-A-Step-by-Step-Lab-Guide

# Network Discovery, Fingerprinting & OSINT Recon — A Step-by-Step Lab Guide

*A hands-on tutorial covering native Windows discovery, active scanning with Nmap and arp-scan on Kali, WAF detection, service/OS fingerprinting, and automated OSINT gathering with Recon-ng.*

---

## Before You Start: Scope and Ethics

Every tool in this guide can be used to map networks and gather intelligence on real organizations without their knowledge. That is exactly why you must only ever run these exercises against:

- Devices and networks you personally own, **or**
- A lab environment you built yourself (e.g., VirtualBox/VMware VMs on an isolated virtual network), **or**
- A target you have **written authorization** to test.

Scanning networks or domains you don't own or have permission to test is illegal in most countries under computer-misuse laws, even if no damage is done. Treat every command below as something to practice in your own lab first.

**Recommended lab setup:**
- A Windows 10/11 VM (or your host machine) on the same virtual network as a Kali Linux VM
- Kali Linux with `nmap`, `arp-scan`, and `recon-ng` installed (all come pre-installed on current Kali images)
- A spare "victim" VM (another Windows or Linux box) to discover and scan
- Internet access from Kali for the OSINT portion of the lab (Recon-ng modules query public web services)

---

## Module 1: Discovering Devices from Windows

Windows ships with built-in tools that reveal what your machine already knows about the local network, with zero extra installs.

### 1.1 Checking your own IP address
```
ipconfig
```
This tells you your machine's own IP, subnet mask, and default gateway — you need this to know which network range you're on before scanning anything.

### 1.2 Reading the ARP cache
The ARP (Address Resolution Protocol) cache is a table Windows keeps of every device it has recently exchanged traffic with on the local network, mapping their IP address to their physical MAC address.
```
arp -a
```
This lists every host your machine has already talked to — a passive way to see what's around you without sending a single new packet. Look at the **Type** column: `dynamic` entries expire after a while; `static` entries are manually configured.

**Follow-up exercise:** Every IP address shown in the `arp -a` output is a live device. Take each one and:
- Ping it: `ping <ip-address>` — confirms it's still up and measures latency
- Resolve its hostname: `nslookup <ip-address>` — see if it has a DNS name
- Try to browse to it if it's a web device: `http://<ip-address>` in a browser

This turns a static list into an active investigation — you're no longer just reading the cache, you're validating and enriching every entry in it.

### 1.3 Viewing shared resources on the network
```
net view /all
```
This queries the Windows network browser service and lists every computer (and its shared folders/printers, thanks to `/all`) that has announced itself on the network — useful for spotting file servers, printers, and other Windows machines that ARP alone won't tell you much about.

> **Limitation to teach your students:** `net view` depends on NetBIOS/SMB browsing being enabled on target machines. Many modern networks disable this, and it will show far fewer devices than an active scan will. That's exactly why the next module — active scanning — exists.

---

## Module 2: Active Discovery from Kali Linux

Where Windows tools mostly read what's already known, Kali's tools actively probe the network to find *everything* that's alive, whether it has volunteered information or not.

### 2.1 `arp-scan` — fast Layer 2 discovery
`arp-scan` sends an ARP request to every address on a subnet and lists everyone who replies. Because it works at Layer 2, it finds devices even if their firewall would normally hide them from a ping.
```
sudo arp-scan --interface=eth0 --localnet
```
- `--interface` — which network card to scan from
- `--localnet` — automatically scans your interface's whole local subnet

You'll get a list of IP address, MAC address, and (where recognizable) the MAC vendor — this last bit is a quick way to spot "that's a Raspberry Pi" or "that's a printer" before you've scanned a single port.

### 2.2 `nmap` — host discovery
```
sudo nmap -sn 192.168.1.0/24
```
`-sn` (previously called a "ping scan") tells Nmap to only check which hosts are up, without port-scanning them. Replace `192.168.1.0/24` with your own subnet from `ipconfig`.

### 2.3 `nmap` — full port and service scan on a discovered host
Once you have a live IP from either tool above:
```
sudo nmap -sV -p- <target-ip>
```
- `-p-` scans all 65,535 ports (slow but thorough — use `-p 1-1000` for a quick pass while learning)
- `-sV` asks Nmap to grab a service **banner** from every open port so you know not just that port 80 is open, but that it's running, say, Apache 2.4.41

**Exercise:** Run `arp-scan` and `nmap -sn` against the same subnet and compare the two device lists side by side. Discuss with students why the counts sometimes differ (e.g., a host that blocks ICMP ping but still replies to ARP).

---

## Module 3: WAF Detection with Nmap

A Web Application Firewall (WAF) sits in front of a website and filters malicious-looking requests. Detecting one tells you a target is likely to block naive attacks and log your activity — important recon before any web testing.

```
nmap --script=http-waf-fingerprint <target>
```
This NSE (Nmap Scripting Engine) script sends a series of requests and inspects the responses — HTTP headers, cookies, and subtle behavioral quirks — against a library of known WAF signatures (Cloudflare, F5 BigIP, Incapsula, ModSecurity, Citrix Netscaler, and others). If a match is found, it prints the WAF's name and, where possible, its version.

For a more thorough (and noisier/slower) pass that sends WAF-specific test payloads:
```
nmap --script=http-waf-fingerprint --script-args http-waf-fingerprint.intensive=1 <target>
```

> **Teaching note:** show students that this is only one detection method. A companion tool called `wafw00f` (also on Kali) is purpose-built for this and often catches WAFs the Nmap script misses — worth a side-by-side comparison as a class exercise: `wafw00f https://target-domain.com`.

---

## Module 4: OS and Service Fingerprinting with Nmap

"Fingerprinting" means figuring out *what* is running on a host — its operating system, and the exact software/version behind each open port — from how it responds to crafted probes.

```
sudo nmap -O --osscan-guess -sV --version-all <target-ip>
```
- `-O` — attempts OS detection by analyzing quirks in how the target's TCP/IP stack responds (things like initial TTL, window size, and how it handles malformed packets)
- `--osscan-guess` — tells Nmap to guess more aggressively when it can't get a perfect OS match
- `-sV` — service/version detection on open ports
- `--version-all` — tries every version-detection probe Nmap has, for maximum accuracy at the cost of speed (lighter alternative: `--version-intensity <0-9>`, where 0 is fastest/least accurate and 9 is slowest/most thorough)

For unidentified services, add:
```
--script fingerprint-strings
```
This prints the raw readable strings Nmap captured from an unrecognized service, letting a human eyeball the response and often identify the software by hand when Nmap's own database can't.

**Discussion point for students:** OS fingerprinting is a *guess* based on protocol behavior, not a certainty — firewalls, load balancers, and virtualization can all distort the fingerprint. Encourage them to run `-O` against a VM whose OS they already know, and see how often Nmap gets it right.

---

## Module 5: Automated OSINT with Recon-ng

Recon-ng is a modular reconnaissance framework — think of it as Metasploit's console style, but aimed at open-source intelligence (OSINT) instead of exploitation: pulling public information about a domain from web-based data sources rather than touching the target directly.

### 5.1 Launching and orienting yourself
```
recon-ng
```
Once inside, the prompt shows your current workspace and module context. The single most useful command as a beginner is:
```
help
```
This lists every root-level command available, and `help <command>` gives documentation for any of them specifically — always the first move when you're unsure of a command's exact syntax.

### 5.2 Creating a workspace
Workspaces isolate the data from each engagement (each "project") into its own database, so results from one domain never mix with another's.
```
workspaces create student-lab
workspaces select student-lab
```

### 5.3 Finding and installing modules — the Marketplace
The marketplace is Recon-ng's module repository. You search it for a capability, then install the specific module you want — you don't get every module by default, which keeps the framework lightweight.

```
marketplace search hackertarget
```
This lists every module whose name or description matches "hackertarget," along with its full path, whether it's already installed, and when it was last updated.

```
marketplace install recon/domains-hosts/hackertarget
```
This downloads and installs that specific module. `hackertarget` is a module that queries the free HackerTarget.com API to pull subdomains/hosts associated with a given domain — a fast first step in mapping an organization's external footprint.

### 5.4 Loading and running the module
```
modules search hackertarget
```
A second, complementary search — this one searches modules you already have installed (rather than the whole marketplace), which is handy once your local module list grows.

```
modules load recon/domains-hosts/hackertarget
```
This loads the module into the current context so you can configure and run it.

```
info
```
Shows the module's description, author, and — most importantly — its required and optional options, so you know exactly what to configure before running it.

```
options set SOURCE example.com
```
`SOURCE` is the option that tells the module *what domain to investigate*. Replace `example.com` with your authorized lab domain.

```
run
```
Executes the module. Results (discovered hosts/subdomains) are automatically saved into your workspace's database — you'll see them again later if you export a report or chain them into another module.

### 5.5 A second module: `interesting_files`
This module checks a target for commonly-exposed sensitive files — things like `robots.txt`, `.git` folders, backup files, or config files left accidentally public — a classic and very cheap OSINT win.

Follow the exact same four-step pattern:
```
marketplace search interesting_files
marketplace install discovery/info_disclosure/interesting_files
modules load discovery/info_disclosure/interesting_files
info
```
Check `info` for the option this module expects (typically the target `SOURCE`/domain, sometimes a specific host list from the `hosts` table populated by your earlier `hackertarget` run), set it, then:
```
run
```

**This is the real power of Recon-ng for your students to notice:** modules can feed each other. The hosts found by `hackertarget` sit in the workspace's `hosts` table, and later modules (like `interesting_files`) can automatically pull from that same table instead of you re-typing every hostname — this is why professional recon is done as a *chain* of modules, not one-off commands.

### 5.6 Reviewing and exporting results
```
show hosts
```
Displays everything the workspace has collected in the hosts table so far, across every module you've run.

```
marketplace search report
```
Search for a reporting module — Recon-ng has several export formats (CSV, HTML, JSON, etc.).
```
marketplace install reporting/csv
modules load reporting/csv
run
```
This writes your findings out to a file, which is what your students should actually hand in as lab evidence, rather than a screenshot of the terminal.

---

## Putting It All Together: A Suggested Lab Flow for Students

1. **Passive Windows recon** — `ipconfig`, `arp -a`, `net view /all` on the student's own VM
2. **Active discovery from Kali** — `arp-scan --localnet` and `nmap -sn` against the same subnet; compare results
3. **Deep scan** one discovered host — `nmap -sV -p-`
4. **Fingerprint it** — `nmap -O --osscan-guess -sV`
5. **WAF check** on any web service found — `nmap --script=http-waf-fingerprint`
6. **OSINT pass in Recon-ng** on an authorized lab domain — workspace → `hackertarget` → `interesting_files` → export a CSV report
7. **Write-up:** each student documents, for every step, *what command they ran, what it found, and what a defender could do to reduce that exposure* (e.g., disabling SMB browsing to blunt `net view`, or blocking ICMP without blocking ARP replies doesn't actually hide a host from `arp-scan`)

That last step is what turns this from "button-pressing" into an actual security lesson: for every discovery technique, there's a corresponding hardening step, and a good student should be able to name both.

---

## Quick Reference Cheat Sheet

| Goal | Windows | Kali |
|---|---|---|
| See your own IP | `ipconfig` | `ip a` |
| See recently-seen hosts | `arp -a` | `ip neigh` |
| Find all hosts on subnet | `net view /all` (passive) | `arp-scan --localnet` / `nmap -sn` |
| Full port + service scan | — | `nmap -sV -p- <ip>` |
| OS fingerprint | — | `nmap -O --osscan-guess <ip>` |
| WAF detection | — | `nmap --script=http-waf-fingerprint <target>` |
| Domain OSINT | — | `recon-ng` (marketplace → modules → run) |
