# Room 1: Passive Recon

**Platform:** TryHackMe
**Room:** https://tryhackme.com/room/passiverecon
**Date completed:** october 2026

## 1. Objective
Learn to gather information about a target from public sources only, without sending traffic to the target. Tools: whois, nslookup, dig, DNSDumpster, crt.sh and Shodan.

## 2. Concepts

### 2.1 Passive vs active reconnaissance
| Type | Meaning | Examples | Detectable? |
|---|---|---|---|
| Passive | Uses public information only; no packets sent to the target | WHOIS, DNS from public resolvers, certificate logs, Shodan, job postings, public GitHub | No |
| Active | Direct interaction with the target | Ping, port scans, probing a web app, social engineering, visiting in person | Yes (logs, IDS/IPS, WAF) |

- Talking to a person from the target company also counts as active, even with no packets involved.
- Active recon without authorisation can cause legal problems. Passive recon is stealthier and is the usual first step.
- Defenders watch their own passive footprint (Shodan alerts, CT log monitoring, DNS change tracking).

### 2.2 WHOIS and RDAP
- WHOIS asks the registrar's server (port 43) about a domain. Command: `whois DOMAIN`.
- Fields: registrar, registrant (often "Withheld for Privacy"), creation / update / expiry dates, name servers, status codes (e.g. `clientTransferProhibited` = domain locked), registrar abuse contact.
- Why an attacker cares: domain age, registrar patterns for phishing, expiry dates for renewal scams, name servers as extra targets, and historical changes (whoxy.com).
- Since 28 January 2025 WHOIS for gTLDs is replaced by RDAP: HTTPS, JSON, better privacy. Example: `curl -s https://rdap.verisign.com/com/v1/domain/DOMAIN | jq .`

### 2.3 DNS lookups: nslookup and dig
Querying public resolvers is still passive because the queries do not go to the target's own servers.

| Record | Meaning |
|---|---|
| A / AAAA | IPv4 / IPv6 address |
| CNAME | Alias pointing to another name |
| MX | Mail servers (lower number = higher priority) |
| SOA | Primary name server, admin email, zone serial |
| TXT | Text data: SPF, DKIM, DMARC, verification strings |

| Tool | Syntax | Example |
|---|---|---|
| nslookup (older) | `nslookup -type=TYPE DOMAIN [SERVER]` | `nslookup -type=MX tryhackme.com 1.1.1.1` |
| dig (preferred) | `dig [@SERVER] DOMAIN TYPE` | `dig @1.1.1.1 tryhackme.com MX` |

- dig is preferred because the output is cleaner, it shows TTL by default and it is better for scripts.
- Using a resolver such as 1.1.1.1 keeps my queries away from my ISP's logs.
- Several A records can mean a CDN or anycast network (e.g. Cloudflare), so each IP must be checked against scope.

### 2.4 Subdomain discovery
- **DNSDumpster:** collects DNS records and draws a map of a domain's hosts.
- **crt.sh (Certificate Transparency logs):** search `%.domain.com` to list certificates issued for subdomains. This is one of the best passive ways to find forgotten subdomains.

### 2.5 Shodan
- A search engine for internet-connected devices (servers, cameras, routers, industrial systems). It stores banners from open ports.
- A host page shows: IP, ASN / organisation, location, open ports, service banners and version strings, tags (e.g. `cdn`, `vuln`).
- Useful filters: `hostname:`, `org:`, `port:`, `country:`, `http.component:`. Censys is a similar alternative.
- Defensive value: finding forgotten or exposed machines before an attacker does.

---

## 3. Practical Work
Commands run from Kali Linux.

| # | Tool | Command | Target | What I found |
|---|---|---|---|---|
| 1 | whois | `whois tryhackme.com` | tryhackme.com | Registered 2018-07-05; registrar NameCheap Inc (IANA ID 1068); name servers kip.ns.cloudflare.com and uma.ns.cloudflare.com (Cloudflare); status clientTransferProhibited; DNSSEC unsigned; registrant hidden by a privacy service |
| 2 | dig | `dig tryhackme.com A` | tryhackme.com | Two A records: 64.239.123.129 and 64.239.109.129 (TTL 243 s). Room 1 curl showed server: Vercel on this IP, so the site is hosted on Vercel while DNS is on Cloudflare |
| 3 | dig | `dig @1.1.1.1 tryhackme.com MX` | tryhackme.com | Five Google MX records: aspmx.l.google.com (priority 1), alt1 and alt2 (5), alt3 and alt4 (10), so email is handled by Google |
| 4 | dig | `dig thmlabs.com TXT` | thmlabs.com | Flag: THM{a5b83929888ed36acb0272971e438d78} |
| 5 | nslookup | `nslookup -type=MX tryhackme.com` | tryhackme.com | Same five Google MX records as dig, shown as "mail exchanger" lines |
| 6 | RDAP | `curl -s https://rdap.verisign.com/com/v1/domain/tryhackme.com \| jq .` | tryhackme.com | Same registration data as JSON: registrar entity 1068, status "client transfer prohibited", link to the Namecheap RDAP service |
| 7 | crt.sh | search `%.tryhackme.com` | tryhackme.com | Subdomains seen: access.vm.tryhackme.com, app.lb-architect.tryhackme.com, assets.tryhackme.com, auth.tryhackme.com, blog.tryhackme.com, blog.vm.tryhackme.com |
| 8 | DNSDumpster | dnsdumpster.com | tryhackme.com |  |
| 9 | Shodan | search `apache`, read the Top Countries and Top Ports facets | Apache | First country: United States / 3rd most common port: 8080 |
| 10 | Shodan | search `nginx`, read the Top Ports facet | nginx | Most common port: 80 |

Task 2 questions (A = active, P = passive): Facebook page of the company = P (public information, no contact with the target); pinging the web server = A (packets are sent to the target); social engineering the IT admin at a party = A (direct interaction with a person from the target). My reasoning: passive means no direct contact with the target, active means any direct interaction.

### 3.1 Extra practice (my own, passive only)
- Second domain tested: example.com
- whois: registrar RESERVED-Internet Assigned Numbers Authority, creation date 1995-08-14T04:00:00Z, name servers elliott.ns.cloudflare.com, hera.ns.cloudflare.com
- dig: A 104.20.23.154, 172.66.147.243, MX 0 ., TXT "v=spf1 -all", "_k2n1y4vw3qtb4skdx9e7dxt97qrmmq9"
- Comparison with tryhackme.com: tryhackme.com uses Cloudflare name servers, Google mail servers and a privacy service that hides the registrant. example.com uses the name servers and mail setup shown above. Both lookups used only public WHOIS and DNS data, so no traffic was sent to either target.

## 4. Tools Used
| Tool | Use |
|---|---|
| whois, curl + jq (RDAP) | Domain registration data |
| dig, nslookup | DNS records |
| crt.sh, DNSDumpster | Subdomain discovery |
| Shodan | Exposed services and device banners |
| Kali Linux | Command line |

## 5. Evidence
Screenshots are in the `ScreenShots` folder and listed in this folder's README.

## 6. Result
Using only public sources, I found that tryhackme.com was registered on 2018-07-05 through NameCheap, uses Cloudflare name servers, receives email through Google mail servers, and resolves to two IP addresses that serve a Vercel-hosted site. The registrant is hidden behind a privacy service and DNSSEC is unsigned. The TXT record of thmlabs.com held the room flag. No traffic was sent to the target at any point.

## 7. Questions I Found Hard
-

## 8. Key Learnings (reword in my own words)
1. Passive recon uses only public sources and sends nothing to the target, so it cannot be detected; active recon touches the target and needs authorisation.
2. WHOIS and DNS records reveal the registrar, name servers, mail provider and IP addresses, which tell an attacker how the organisation is built.
3. Certificate transparency logs and DNSDumpster expose subdomains, including forgotten ones that may be unpatched.
4. Shodan shows what the internet already knows about a server (ports, software versions), so defenders should check their own footprint first.
