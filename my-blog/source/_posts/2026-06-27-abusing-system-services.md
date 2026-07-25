---
layout: post
title: "Persistence by Abusing systemd Services"
date: 2026-06-27 19:30:00 +0200
description: "A short showcase of user-level persistence through systemd user services on a Linux host."
categories: [security]
tags: [security]
---

# Persistence by Abusing systemd Services

User-level systemd services live in:

```text
~/.config/systemd/user/
```

If the directory does not exist yet, create it first:

```bash
mkdir -p ~/.config/systemd/user
```

Create a user service file:

```bash
vi ~/.config/systemd/user/persistence.service
```

```ini
[Unit]
Description=Persistence service

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo pwned'
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

Reload the user systemd manager and enable the service:

```bash
systemctl --user daemon-reload
systemctl --user enable persistence.service
```

`enable` does not run the service immediately. It creates the user-level autostart link, so the service starts the next time the user's `default.target` is reached. In most setups, that means the next login for that user.

To start it immediately as well:

```bash
systemctl --user start persistence.service
```

This provides persistence across logins and reboots where the user logs in again. If the service should start after boot without an interactive login, lingering must be enabled for that account:

```bash
loginctl enable-linger "$USER"
```
