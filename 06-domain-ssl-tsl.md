# Domains, DNS & SSL/TLS

A domain name is the human-readable address people type to reach your server, DNS is the system that translates that name into your server's actual numeric address, and SSL/TLS is what encrypts the connection so it's safe to use.

## Table of Contents

- [What Is a Domain Name?](#what-is-a-domain-name)
- [What Is an IP Address?](#what-is-an-ip-address)
- [How Domains and IPs Relate](#how-domains-and-ips-relate)
- [What Is DNS?](#what-is-dns)
- [DNS Record Types You Need to Know](#dns-record-types-you-need-to-know)
- [Buying a Domain](#buying-a-domain)
- [Pointing a Domain at Your Server](#pointing-a-domain-at-your-server)
- [How DNS Propagation Works](#how-dns-propagation-works)
- [Subdomains Explained](#subdomains-explained)
- [What Is SSL/TLS?](#what-is-ssltls)
- [HTTP vs HTTPS](#http-vs-https)
- [Getting a Free SSL Certificate with Let's Encrypt](#getting-a-free-ssl-certificate-with-lets-encrypt)
- [Auto-Renewal of Certificates](#auto-renewal-of-certificates)
- [Common Mistakes Beginners Make](#common-mistakes-beginners-make)
- [Quick Revision](#quick-revision)

## What Is a Domain Name?

- A **domain name** is the human-friendly address of a website — e.g., `yourapp.com` — that people type into a browser instead of a long string of numbers.
- Domains are organized hierarchically: `yourapp.com` is a **second-level domain** under the **top-level domain (TLD)** `.com`. Other common TLDs include `.dev`, `.io`, `.net`, `.org`.
- You don't "own" a domain outright — you **rent** it, typically year by year, from a **domain registrar** (a company authorized to sell domain registrations).

## What Is an IP Address?

- An **IP address** is the actual numeric address of a computer on the internet — the real destination your browser needs to know to connect to a server.
- **IPv4** addresses look like `203.0.113.25` (four numbers 0–255, separated by dots). **IPv6** addresses are longer, like `2001:0db8:85a3::8a2e:0370:7334`, created because IPv4 addresses are running out globally.
- When you rent a VPS, the hosting provider assigns your server a **static (unchanging) public IP address** — this is the actual location DNS will point your domain to.

> 💡 **Tip:** Think of an IP address like GPS coordinates for a building, and a domain name like the building's street address/name. Both point to the same place, but humans remember and type names, not coordinates.

## How Domains and IPs Relate

```text
User types: yourapp.com
        │
        ▼
   DNS lookup translates the domain into an IP address
        │
        ▼
Browser connects to: 203.0.113.25 (your VPS)
```

- Without DNS, you'd have to make every visitor type your server's raw IP address to reach your site — technically possible, but obviously impractical and unmemorable.

## What Is DNS?

- **DNS (Domain Name System)** is essentially the internet's phonebook — a distributed system of servers that store the mapping between domain names and IP addresses, and answer lookup requests from browsers all over the world.
- When you buy a domain, you configure **DNS records** for it, telling the internet "this domain name maps to this IP address" (and other information, covered below).
- DNS records are typically managed through your domain registrar's dashboard, or through a separate DNS provider (e.g., Cloudflare) if you choose to use one instead.

## DNS Record Types You Need to Know

| Record Type | Purpose | Example |
|---|---|---|
| **A** | Points a domain (or subdomain) to an **IPv4** address | `yourapp.com → 203.0.113.25` |
| **AAAA** | Points a domain to an **IPv6** address | `yourapp.com → 2001:0db8::1` |
| **CNAME** | Points a domain/subdomain to *another domain name*, not an IP directly | `www.yourapp.com → yourapp.com` |
| **MX** | Directs email for the domain to a mail server | Used for setting up `you@yourapp.com` email |
| **TXT** | Stores arbitrary text, often used for domain verification or security (e.g., SPF for email) | Verification codes, etc. |
| **NS** | Specifies which nameservers are authoritative for the domain | Usually set automatically by your registrar |

- **A record** is the one you'll use constantly for basic deployment — it's the direct link between your domain and your server's IP address.
- **CNAME** is useful for subdomains that should just mirror another domain, rather than needing their own separate IP mapping.

> ⚠️ **Warning:** A common beginner mistake is creating both an A record and a CNAME record for the exact same name — this is invalid and causes DNS errors. A given hostname should have one or the other, not both.

## Buying a Domain

- You purchase domains through a **domain registrar** — common ones include Namecheap, Google Domains (now migrated to Squarespace), Cloudflare Registrar, and GoDaddy.
- Pricing varies by TLD: `.com` is usually the most affordable and universally recognized; newer TLDs like `.dev` or `.app` can have different pricing and sometimes stricter security requirements (e.g., `.dev` requires HTTPS by default).
- Domains are rented on a subscription basis — typically renewed yearly. Losing track of renewal can mean losing your domain entirely if it expires and someone else registers it.

> 💡 **Tip:** When we do the real-world walkthrough at the end of this series, we'll actually purchase a domain together, step by step, using a specific registrar.

## Pointing a Domain at Your Server

Once you own a domain and have a VPS with a static IP, you connect them by creating DNS records in your registrar's DNS management dashboard:

```text
Type: A
Name: @              (means "the root domain itself", i.e. yourapp.com)
Value: 203.0.113.25  (your server's IP address)
TTL: Automatic (or 3600 seconds)

Type: A
Name: www
Value: 203.0.113.25
TTL: Automatic
```

- `@` — a common shorthand meaning "the domain itself with no subdomain prefix" (`yourapp.com`, not `something.yourapp.com`).
- `www` — a second A record so that `www.yourapp.com` also resolves correctly (many users habitually type the `www.` prefix).
- **TTL (Time To Live)** — how long (in seconds) other DNS servers are allowed to cache this record before checking for updates again. Lower TTL means changes propagate faster but is queried more often; higher TTL is more efficient but slower to update.

## How DNS Propagation Works

- **DNS propagation** is the delay between updating a DNS record and that change being visible everywhere on the internet.
- This happens because DNS results are **cached** at many layers — your ISP, your operating system, your browser, and DNS servers around the world — each holding onto old answers until their TTL expires.
- Propagation can take anywhere from a few minutes to (rarely) up to 48 hours, though modern setups are usually fast (minutes) if TTLs are set reasonably low.

> 💡 **Tip:** You can check propagation status using a tool like `whatsmydns.net`, which shows what different DNS servers around the world are currently returning for your domain — useful for confirming a change has taken effect globally.

## Subdomains Explained

- A **subdomain** is a prefix added before your main domain, used to organize different parts of your infrastructure — e.g., `api.yourapp.com`, `admin.yourapp.com`, `blog.yourapp.com`.
- Each subdomain is just another DNS record (usually an A or CNAME record) and can point to the *same* server as your main domain, or to a completely different one.
- This is exactly how a real full-stack deployment often separates concerns: `yourapp.com` for the frontend, `api.yourapp.com` for the backend, `admin.yourapp.com` for the admin panel — even if all three are technically the same VPS, routed by Nginx `server_name` blocks (from deploy-04).

```nginx
# Example: separate Nginx server blocks per subdomain, same server
server {
    listen 80;
    server_name api.yourapp.com;
    location / { proxy_pass http://localhost:4000; }
}

server {
    listen 80;
    server_name admin.yourapp.com;
    location / { proxy_pass http://localhost:5000; }
}
```

## What Is SSL/TLS?

- **SSL (Secure Sockets Layer)** and its modern successor **TLS (Transport Layer Security)** are protocols that encrypt the connection between a browser and a server, so data (passwords, form submissions, cookies) can't be read or tampered with by anyone intercepting the traffic.
- People still say "SSL" out of habit/legacy naming, but modern encrypted connections actually use **TLS** under the hood — the terms are used almost interchangeably today.
- An **SSL/TLS certificate** is a small file, issued by a trusted authority, that proves a domain's identity and enables this encryption. It's what makes the padlock icon appear in a browser's address bar.

## HTTP vs HTTPS

| | HTTP | HTTPS |
|---|---|---|
| **Encryption** | None — data sent in plain text | Encrypted via SSL/TLS |
| **Port** | 80 | 443 |
| **Browser indicator** | "Not Secure" warning | Padlock icon |
| **SEO / trust impact** | Search engines and users penalize/distrust it | Expected as standard today |
| **Required for** | Nothing modern | Payment forms, logins, modern browser APIs, SEO |

> ⚠️ **Warning:** Serving login forms, payment details, or any sensitive user data over plain HTTP is a serious security risk — anyone on the same network (e.g., public WiFi) could intercept that data in transit. HTTPS is considered mandatory for any real production app today, not optional polish.

## Getting a Free SSL Certificate with Let's Encrypt

- **Let's Encrypt** is a free, automated, widely trusted certificate authority that issues SSL/TLS certificates at no cost — it's the standard choice for most personal and small-to-mid production deployments today.
- The easiest way to use it on a server running Nginx is via a tool called **Certbot**, which automates requesting, installing, and configuring the certificate for you.

```bash
# Install Certbot and its Nginx plugin
sudo apt update
sudo apt install certbot python3-certbot-nginx

# Request and automatically configure a certificate for your domain
sudo certbot --nginx -d yourapp.com -d www.yourapp.com
```

- `-d yourapp.com -d www.yourapp.com` — specifies which domain(s) the certificate should cover; you can list multiple.
- Certbot will automatically edit your Nginx config to listen on port `443`, load the new certificate, and set up a redirect from HTTP to HTTPS — all in one command.
- After this runs successfully, `https://yourapp.com` will show a valid padlock in the browser.

> 💡 **Tip:** Certbot's `--nginx` plugin is specifically aware of Nginx's configuration format and edits it directly for you — this is a huge convenience compared to manually configuring SSL certificate paths by hand.

## Auto-Renewal of Certificates

- Let's Encrypt certificates are only valid for **90 days**, by design, to encourage automated renewal practices rather than long-lived, potentially-forgotten certificates.
- Certbot automatically sets up a scheduled task (via `systemd timer` or `cron`) to renew certificates before they expire, with no manual action needed from you.

```bash
# Test that renewal works correctly, without actually renewing yet
sudo certbot renew --dry-run
```

> ⚠️ **Warning:** If a certificate silently fails to renew (e.g., due to a firewall change or DNS misconfiguration) and expires, your site will suddenly show a scary "Your connection is not private" warning to all visitors. Periodically running `certbot renew --dry-run` is a good habit to confirm auto-renewal is still healthy.

## Common Mistakes Beginners Make

- Forgetting to add the `www` A record, so `www.yourapp.com` doesn't resolve even though the bare domain does.
- Expecting DNS changes to be instant and panicking when they take a few minutes to propagate.
- Creating conflicting A and CNAME records for the same hostname.
- Running Certbot before DNS has actually propagated — certificate issuance requires the domain to already correctly point to your server.
- Assuming SSL setup is a "nice to have" and delaying it — modern browsers actively warn users away from HTTP-only sites.

## Quick Revision

- A domain name is a human-readable address that DNS translates into your server's actual IP address.
- The A record is the core DNS record type connecting a domain (or subdomain) directly to a server's IP address.
- Domains are rented yearly from a registrar; DNS records are configured in that registrar's dashboard (or a separate DNS provider like Cloudflare).
- DNS propagation is the caching-related delay before a DNS change is visible everywhere — usually minutes, rarely up to 48 hours.
- Subdomains (`api.yourapp.com`, `admin.yourapp.com`) let you cleanly separate frontend, backend, and admin panel, even on a single server, using Nginx `server_name` blocks.
- SSL/TLS encrypts the connection between browser and server; HTTPS (port 443) is now considered mandatory for any real production app.
- Let's Encrypt, via the Certbot tool, provides free, automated SSL certificates and can configure Nginx for HTTPS in a single command.
- Let's Encrypt certificates expire every 90 days by design, but Certbot sets up automatic renewal so you rarely need to think about it.