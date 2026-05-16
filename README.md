# Vulnhub | digitalworld.local - development Walkthrough

An in-depth, systematic walkthrough demonstrating the complete compromise of the **Development** boot-to-root machine from Vulnhub. This lab showcases tactical web enumeration, traffic packet analysis, restricted environment breakouts, and vertical privilege escalation.

---

## 🎯 Lab Objectives
* **Advanced Reconnaissance**: Identifying concealed directories and bypassing basic deception barriers.
* **Network Forensics**: Inspecting captured traffic files to extract sensitive structural hints and user activity.
* **Restricted Shell Breakout**: Escaping a hardened Python-based limited shell environment (`lshell`).
* **Privilege Escalation**: Exploiting loose Sudo permission models on administrative text editors (`GTFOBins`).

---

## 🧭 Exploitation Methodology

### 1. Information Gathering & Service Scanning
An initial thorough network scan using `Nmap` identified several open doors:
* **Port 22/tcp**: OpenSSH 7.6p1
* **Ports 139 & 445/tcp**: Samba smbd (Ubuntu)
* **Port 8080/tcp**: HTTP Web Proxy Server configured with a deceptive banner claiming to be `Microsoft IIS 6.0` (while actually running on Linux).

An aggressive automated fuzzing attempt via `Gobuster` triggered defensive rate-limiting/blacklisting mechanisms on the host's intrusion detection configuration. However, manual index inspection guided investigation towards the `/html_pages/` sub-directory.

### 2. Directory Traversal & Deep Enumeration
By analyzing the source files uncovered inside the server mapping, a systematic traversal strategy was used across extensions (`.html` vs extensionless paths). Significant technical findings included:
* **`development.html`**: Contained an HTML comment explicitly pointing to a critical structural endpoint: `./developmentsecretpage`.
* **Binary Disclosures**: Pages like `default.html` and `register.html` contained raw binary streams. Decoding the `register.html` stream revealed the hint: *"Surely development secret page is not that hard to find?"*
* **The Log Leak**: Deeper directory searching inside `/developmentsecretpage/` exposed an exposed plaintext user registry configuration file at `/developmentsecretpage/slog_users.txt`.

### 3. Offline Credential Cracking & Initial Access
The exposed `slog_users.txt` file utilized an legacy PHP text-file login library module, leaking raw system usernames and cryptographic `MD5` hashes:
```text
admin, 3cb1d13bb83ffff2defe8d1443d3a0eb 
intern, 4a8a2b374f463b7aedbb44a066363b81 
patrick, 87e6d5ce79af90dbe07d387d3d0579e 
qiu, ee64497098d0926d198f54f6d5431f98
