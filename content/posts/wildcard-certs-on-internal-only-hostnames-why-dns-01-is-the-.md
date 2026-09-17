---
title: "Wildcard Certs on Internal-Only Hostnames: Why DNS-01 Is the Only Real Option"
date: 2026-09-17
description: "How to get valid, trusted HTTPS certificates for LAN-only homelab services using the ACME DNS-01 challenge, without exposing anything to the internet."
tags: ["dns", "tls", "reverse-proxy", "self-hosted"]
---

If every internal service in your homelab is sitting behind self-signed certs, you've probably trained yourself to click through browser warnings without reading them. That's a bad habit to build, and it's also completely unnecessary. You can get real, publicly trusted certificates for hostnames that never resolve outside your LAN. The catch is that the usual HTTP-01 challenge can't do it, so you need DNS-01.

## Why HTTP-01 doesn't work here

HTTP-01 is how most people first get a Let's Encrypt cert: the ACME client puts a token at a well-known URL on port 80, and the CA fetches it over the public internet to prove you control the domain. That works fine for a public-facing web server, but it assumes the CA can actually reach your host.

For something like `proxmox.example.com` that only resolves on your internal DNS server, there's nothing for the CA to fetch. Port forwarding just to satisfy a cert challenge is a bad trade: you're opening a hole in your firewall for a service that was never meant to be public, purely so a validation request can land.

## What DNS-01 actually checks

DNS-01 proves domain ownership a different way: the ACME client creates a TXT record at `_acme-challenge.yourhostname.example.com` with a value the CA gives it, and the CA checks that record over public DNS. It never touches your server at all. As long as your domain's authoritative DNS is publicly queryable (which it almost always is, even for a domain you only use internally), this works regardless of where the service itself lives.

This also unlocks wildcard certificates, which HTTP-01 can't issue at all. A single cert for `*.internal.example.com` covers every LXC, VM, and container behind your reverse proxy without provisioning a new one per hostname.

## Getting the API token scoping right

DNS-01 requires your ACME client to create and delete TXT records programmatically, which means it needs API credentials for your DNS provider. This is the part people get sloppy with. Don't hand your automation a full-account API key if the provider supports scoped tokens.

At minimum, look for:

- A token restricted to a single zone, not your whole DNS account
- Permissions limited to DNS record edit (not domain transfer, not billing, not registrar controls)
- A token you can rotate independently of any personal account credentials

If your provider only offers all-or-nothing API keys, keep that token confined to the one box running the ACME client and don't reuse it anywhere else. That client now has the ability to prove ownership of your domain, so treat the credential accordingly.

## Picking a client

- **acme.sh** has the widest range of built-in DNS provider integrations and is a reasonable default if you're issuing certs on a general-purpose box and distributing them yourself.
- **Caddy** with the matching DNS provider plugin handles issuance and renewal automatically as part of normal operation, which is convenient if Caddy is already your reverse proxy.
- **Traefik** supports DNS-01 challenges natively per provider and fits well if you're already running it for container routing.

Whichever you pick, confirm it supports your specific DNS provider's API before committing. Not every provider has a stable, well-maintained plugin, and a flaky one will cause silent renewal failures months later when you've forgotten how any of this works.

## Gotchas worth knowing up front

**CAA records.** If your domain has a CAA record restricting which CAs can issue for it, make sure Let's Encrypt (or whichever CA you're using) is actually allowed. A missing or overly strict CAA record causes issuance to fail with an error that doesn't obviously point at CAA as the cause.

**DNS propagation delays.** The CA checks the TXT record shortly after your client creates it. If your DNS provider or any caching resolver in the path is slow to propagate, the challenge can fail even though the record is technically correct. Most clients have a configurable wait/verify step for this; don't skip it just to speed up first-time setup.

**Rate limits.** Let's Encrypt's wildcard and duplicate-certificate rate limits are per registered domain, not per subdomain. If you're testing configuration and re-issuing repeatedly, use the staging environment first so you don't burn your production rate limit budget on typos.

**Renewal automation still needs monitoring.** A cert that auto-renews via DNS-01 can still fail silently if the API token expires, the DNS provider changes their API, or the zone gets migrated. Point whatever alerting you already run at certificate expiry dates, the same way you'd watch a backup job. An automated renewal you never check on is just a slower way to get the same expired-cert page you were trying to avoid.
