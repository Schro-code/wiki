---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# DNS Records — Custom Domains in App Service

## What it is

The types of DNS records used to verify ownership of a domain and redirect traffic to Azure App Service.

## What problem it solves

To serve an App Service app from your own domain (e.g. `www.mycompany.com`) instead of `*.azurewebsites.net`, Azure must be sure you own the domain, and traffic must be routed to the app. A, CNAME and TXT records cover routing and ownership verification.

## Notes

### Record types

#### A record

Points a domain directly to an **IP**.

```
mycompany.com  →  20.30.40.50
```

Problem: if Azure changes the IP (due to maintenance or scaling), the record becomes outdated.

#### CNAME record

Points a domain to **another domain** (alias). Recommended for subdomains (`www`).

```
www.mycompany.com  →  myapp.azurewebsites.net
```

Azure handles the final resolution. If the IP changes behind the scenes, the CNAME is unaffected.

> A CNAME cannot be used for the root domain (`@`). For `mycompany.com` without `www` you have to use an A record.

#### TXT record

Free text. It does not affect traffic — it serves exclusively for **ownership verifications**.

```
mycompany.com  →  "azure-site-verification=abc123xyz"
```

If you can write to the domain's DNS, Azure assumes you are the owner.

### Flow to add a custom domain in App Service

```
1. Azure generates a unique verification code for your app
2. You go to your registrar (GoDaddy, Namecheap, etc.) and add two records:
   - TXT with the code  →  proof of ownership
   - CNAME: www.mycompany.com → myapp.azurewebsites.net  →  redirects the traffic
3. Azure checks the TXT, validates the domain and enables it
4. (Optional) You assign a TLS certificate — free with App Service Managed Certificate
```

### Summary

| Record | Who generates it | What it is for |
|----------|-----------|----------------|
| A | You | Point the root domain to a fixed IP |
| CNAME | You | Subdomain alias to azurewebsites.net |
| TXT | Azure (code) + you (publish it) | Verify domain ownership |

## Relations

- [[azure-container-services]] — App Service is the hosting option these custom domains attach to

## Tags

#azure #az204
