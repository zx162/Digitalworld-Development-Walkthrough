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

Using John the Ri<img width="1056" height="348" alt="image" src="https://github.com/user-attachments/assets/767f6db3-f984-40d8-a03a-73ac268303c2" />
<img width="1133" height="348" alt="image" src="https://github.com/user-attachments/assets/708d20ca-608e-4179-8b0c-13e0efaf4e8e" />
pper with the standard rockyou.txt dictionary format, the hash for the intern user was successfully cracked, yielding the plaintext credential: 12345678900987654321.

Establishing an initial foothold was accomplished via SSH:

Bash
ssh intern@<TARGET_IP>
4. Escaping the Hardened Shell (lshell Breakout)
Upon successful SSH authentication, the environment was restricted inside Limited Shell (lshell), stripping away core commands like id and whoami.

Vulnerability Identification: Querying the environment (echo $SHELL) verified the lshell restriction layer. Running the standard help command mapped out the strictly whitelisted actions: cd, clear, echo, exit, help, ll, lpath, ls.

The Breakout: Because the native echo function was poorly sanitized, it allowed direct arbitrary execution of inline Python runtime processes.

Payload Execution:

Plaintext
intern:~$ echo os.system('/bin/bash')
Executing this statement crashed the restriction logic and spawned a fully functional, standard native /bin/bash instance.

5. Local Lateral Movement & Privilege Escalation
With a clean interactive terminal operational:

Checked local server artifacts and discovered back up files indicating password reuse dependencies across administrative operations.

Pivoted laterally from the intern user context into the patrick account using the cracked credential string extracted earlier (P@ssw0rd25).

Evaluated available execution contexts by issuing:

Bash
patrick@development:~$ sudo -l
The configuration revealed absolute password-exempt Sudo clearance (NOPASSWD) for two native administrative binaries: /usr/bin/vim and /bin/nano.

Final Root Exploitation: Utilizing GTFOBins methodology, launching sudo vim or sudo nano executes the binary with full implicit root context. Inside sudo nano, initiating the file/command pipeline buffer (Ctrl+R then Ctrl+X) allowed the execution of a system command execution interface payload:

Plaintext
reset; sh 1>&0 2>&0
This dropped execution parameters back into a terminal prompt running under absolute root security configurations.

🏁 Root Level Access Achieved (uid=0) & Flags Captured!

🛠️ Defensive Remediation Recommendations
Remove Deceptive Insecure Configuration Details: Avoid reliance on misleading server-header tags (e.g., masking Linux behind IIS 6.0) as they do not hinder determined human attackers.

Secure Sensitive Flat-File Registries: Never store application users or database login strings in cleartext folders or configurations accessible within the active web application paths.

Impose Strict Sudo Principle of Least Privilege: Text editors capable of spawning sub-processes (nano, vim) must never be granted execution rights under password-exempt root environments (NOPASSWD).
