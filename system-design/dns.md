---
domain: system-design
tags: [system-design]
sources: []
updated: 2026-09-01
---

# DNS — Domain Name System

## What it is

DNS is the system that translates human-readable domain names (`google.com`) into IP addresses (`142.250.185.46`). It is a distributed, hierarchical database — no single server knows everything, but together they can resolve any name.

## What problem it solves

Devices communicate via IP addresses, but humans use names. DNS bridges that gap so you never have to know (or hardcode) an IP to reach a service.

## Notes

### The resolution chain — four servers

Every DNS query travels through up to four types of servers in sequence:

```
Browser
  → DNS Resolver       (your librarian — does all the work)
    → Root Server      (says: ".com? go to the .com TLD server")
      → TLD Server     (says: "google.com? its authoritative server is X")
        → Authoritative Server  (says: "google.com = 142.250.185.46")
  ← Resolver caches and returns the answer
```

| Server | Role |
|---|---|
| **DNS Resolver** | Run by your ISP or a public provider (e.g. `8.8.8.8`). Receives your query, walks the chain, returns the answer. Caches results. |
| **Root Server** | 13 types, hundreds of copies worldwide. Doesn't know IPs — only knows which TLD server handles each extension (`.com`, `.org`, `.es`). |
| **TLD Server** | Manages one extension. Knows which authoritative server is responsible for each domain under it. Managed by IANA/ICANN. |
| **Authoritative Server** | Controlled by the domain owner. Holds the actual records. Final answer. |

### Query types — who does the work

| Type | How it works | When |
|---|---|---|
| **Recursive** | You ask the resolver once; it does everything and returns the answer | Normal browser query |
| **Iterative** | Resolver returns a referral; your client must follow it step by step | DNS tools like `dig` |
| **Non-recursive** | Answer is already cached — no lookup needed | Most repeat queries |

In practice: browsers use recursive (ask once, get the answer). The resolver uses iterative internally when walking the chain.

### Record types — DNS is a database

The authoritative server holds records. Each type answers a different question:

| Record | Question | Example |
|---|---|---|
| **A** | IPv4 address for this name? | `google.com → 142.250.185.46` |
| **AAAA** | IPv6 address for this name? | `google.com → 2607:f8b0::...` |
| **CNAME** | Is this an alias for another name? | `www.google.com → google.com` |
| **MX** | Where does email for this domain go? | `@gmail.com → mail.google.com` |
| **NS** | Which server is authoritative for this domain? | Delegation pointer |
| **TXT** | Free text — admin notes, email security tokens | SPF, DKIM verification |
| **PTR** | Reverse lookup: IP → domain name | Used for spam filtering |
| **SRV** | What port does a specific service run on? | Used in some service discovery setups |

**TTL (Time to Live):** every record has one — how many seconds a cache should keep the answer before asking again. Low TTL = fast propagation but more queries. High TTL = fewer queries but slow to update.

### CNAME in practice

When you deploy to Vercel, Heroku, or Cloudflare, they ask you to create a CNAME pointing your domain to their platform:

```
yourdomain.com → CNAME → yourapp.vercel.app
```

Their IP can change; the alias always follows. You never update your DNS again.

## Developer relevance

| Situation | What matters |
|---|---|
| Deploy to Vercel/Heroku/Cloudflare | Create a CNAME record — don't hardcode their IP |
| Custom domain for your app | A record pointing to your server's IP |
| Email deliverability | MX + TXT (SPF/DKIM) records on your domain |
| Microservices | Services register a name; DNS resolves to whatever IP is current — this is the basis of service discovery |
| `dig google.com` | CLI tool to manually walk a DNS resolution and debug it |
| TTL matters at deploy time | Set low TTL before a migration, restore after — controls how fast the world sees your change |

## Relations

- [[ip-addresses]] — DNS resolves names to IP addresses; without IP you can't communicate once DNS answers

## Tags

#system-design
