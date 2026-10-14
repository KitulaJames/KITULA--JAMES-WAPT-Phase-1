# Room 3: Content Discovery

**Platform:** TryHackMe
**Room:** https://tryhackme.com/room/contentdiscoveryx
**Date completed:** 13 October 2026

## 1. Objective
Learn to find content on a web server that is not linked or meant to be public (hidden pages, directories, files, admin portals, backups, subdomains), using manual techniques, OSINT and automated tools. Target: the room's "Acme IT Support" lab machine.

## 2. Concepts

### 2.1 What content discovery is
Content means anything hosted on a web server: pages, files, directories, admin portals, config files and backups. Finding hidden content is a core part of every web pentest, and the results feed later stages of testing.

| Approach | Techniques |
|---|---|
| Manual | robots.txt, sitemap.xml, favicon fingerprinting, HTTP headers, framework stack |
| OSINT | Google dorking, Wappalyzer, Wayback Machine, GitHub, S3 buckets |
| Automated | Gobuster dir, dns and vhost modes |

### 2.2 Manual discovery
- **robots.txt:** tells crawlers which paths not to index. Owners often list sensitive paths here, so it is a ready-made list of interesting locations. It is only a request to bots, not a security control, so a listed path can still be opened directly.
- **sitemap.xml:** lists the pages the owner wants indexed. It can include hidden or staging URLs (here `/customers/login` and a secret area) and parameters such as `?id=` that hint at input points to test.
- **HTTP headers:** `curl http://IP -v` shows the response headers. `Server` and `X-Powered-By` reveal the software and framework, and a custom header can even hold a flag.
- **Framework stack:** the framework name can come from the favicon, headers, or comments and copyright notices in the page source. Its documentation often gives default admin paths and default credentials, which is why default credentials such as admin/admin must be changed.
- **Favicon:** the default favicon of a framework can identify it. Hashing the favicon and comparing it to a database of known favicons reveals the technology.

### 2.3 OSINT discovery
OSINT (open-source intelligence) uses free public tools, so nothing is sent to the target.

| Technique | What it is | What it can reveal |
|---|---|---|
| Google dorking | Google's advanced search operators combined to filter results | Exposed admin panels, leaked documents, login pages |
| Wappalyzer | Browser extension and online tool that identifies a site's technologies | Frameworks, CMS, CDN, analytics and often version numbers |
| Wayback Machine | Internet archive (web.archive.org) | Pages removed from the live site: old login forms, forgotten endpoints |
| GitHub | Cloud platform for Git repositories | API keys, credentials, config and .env files; check the commit history too |
| S3 buckets | Amazon cloud storage, URL format https://{name}.s3.amazonaws.com | Files in misconfigured, public buckets |

Dork operators: `site:` (one domain), `inurl:`, `filetype:`, `intitle:`, `intext:`, `cache:`. They can be combined, for example `site:tryhackme.com filetype:pdf`.
Common S3 bucket names to try: `{company}-assets`, `{company}-backup`, `{company}-www`, `{company}-dev`.

### 2.4 Automated discovery with Gobuster
Automated tools send hundreds or thousands of requests built from a wordlist to see which paths exist. Wordlists: SecLists (`Discovery/Web-Content/common.txt`, `directory-list-2.3-medium.txt`).

| Mode | Purpose | Example |
|---|---|---|
| dir | Brute-force directories and files | `gobuster dir -u http://IP -w WORDLIST` |
| dns | Brute-force subdomains through DNS | `gobuster dns -d example.thm -w WORDLIST --wildcard` |
| vhost | Find virtual hosts by changing the Host header | `gobuster vhost -u http://IP --domain example.thm -w WORDLIST --append-domain --exclude-length 250-320` |

| Flag | Meaning |
|---|---|
| -u | Target URL (dir and vhost) |
| -d | Target domain (dns mode) |
| -w | Wordlist (required) |
| -t | Threads (default 10) |
| -x | File extensions, e.g. `.php,.txt` |
| -s | Only show these status codes |
| -o | Write output to a file |
| --delay | Wait between requests (rate-limited servers) |
| --wildcard | Force dns results even with wildcard DNS |

- **Subdomain vs virtual host:** a subdomain is resolved by DNS (a DNS record). A virtual host is resolved by the web server from the `Host:` header, so several sites can share one IP.
- **Reading dir results:** 200 = exists, 301 and 302 = redirect, 404 = not found. A 302 to a login page (`/customers`) shows a protected area. Files such as `development.log` can hold sensitive data.
- **Responsible use:** these tools send thousands of requests, so they must only be run against machines I am authorised to test (here, the room's lab machine).

---

## 3. Practical Work
All commands were run from my Kali machine against the room's lab machine (IP changes each time the VM starts).

| # | Task | Command or action | What I found |
|---|---|---|---|
| 1 | robots.txt | `curl http://TARGET/robots.txt` | Disallowed directory: /staff-portal (the file allows everything else with Allow: /) |
| 2 | sitemap.xml | `curl http://TARGET/sitemap.xml` | Secret area path: /s3cr3t-area (also lists /customers/login and /news/article?id=1,2,3) |
| 3 | Headers | `curl http://TARGET -v` | X-FLAG header value: THM{HEADER_FLAG} ; also Server: nginx/1.18.0 (Ubuntu) and X-Powered-By: THM-Framework |
| 4 | Framework stack | Followed the framework link from the page source comment, found the admin path in its documentation, logged in with the default admin / admin | Admin portal path: /thm-framework-login ; flag: THM{CHANGE_DEFAULT_CREDENTIALS} (the framework documentation named the path, and the default admin / admin login worked) |
| 5 | Gobuster dir | `gobuster dir -u http://TARGET -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt` | Directory starting with /mo: /monthly ; log file: /development.log (also found /private, /assets, /customers redirecting to /customers/login, /robots.txt and /sitemap.xml) |
| 6 | Gobuster dns | `gobuster dns -d example.thm -w SUBDOMAIN_LIST --wildcard` | Required shorthand flag besides dns and -w: -d (the domain) ; subdomains found: not run against the lab (the dns mode needs the room's DNS setup); I used vhost mode instead, see the next row |
| 7 | Gobuster vhost | `gobuster vhost -u http://TARGET --domain acmeitsupport.thm -w SUBDOMAIN_LIST --append-domain` | 3 virtual hosts answered 200: blog.acmeitsupport.thm, admin.acmeitsupport.thm and shop.acmeitsupport.thm (found with --append-domain and --exclude-length 250-320) |
| 8 | OSINT questions | Google dork, Wappalyzer, Wayback Machine, S3 | Dork operator that limits results to one site: site ; technology tool and extension: Wappalyzer ; Wayback Machine address: https://web.archive.org/ ; S3 bucket URL ending: .s3.amazonaws.com |

**Observations from the Acme IT Support site (own words):** the response headers leaked the web server (nginx/1.18.0 on Ubuntu), the framework (THM-Framework) and a custom X-FLAG header. The page source had a comment naming the framework, whose documentation gave the admin portal path, and the default admin / admin login still worked. Gobuster dir found /assets, /contact, /customers (302 to /customers/login), /development.log, /monthly, /news, /private, /robots.txt and /sitemap.xml, and vhost mode found three hidden virtual hosts (blog, admin, shop).

### 3.1 Extra practice (own work, room target only)
- `curl -I http://TARGET` compared with `curl -v`: curl -v (GET) returned 200 OK and the page, while curl -I (HEAD) returned 404 Not Found, so the server treats HEAD and GET differently, but both responses still leaked Server, X-FLAG and X-Powered-By
- Status of a path found by Gobuster: /customers returned 302 and redirected to /customers/login, which shows a protected area behind a login page

## 4. Tools Used
| Tool | Use |
|---|---|
| curl | View robots.txt, sitemap.xml and response headers |
| Browser and page source | Find comments, the framework link and the admin portal |
| Gobuster (dir, dns, vhost) | Automated discovery of paths, subdomains and virtual hosts |
| SecLists wordlists | Candidate names for the scans |
| Kali Linux | Command line |

## 5. Evidence
Screenshots are in the `ScreenShots` folder and listed in this folder's README.

## 6. Result
Manual checks (robots.txt, sitemap.xml, headers, source comment, framework documentation) exposed hidden paths, the software stack and an admin portal that accepted the default admin / admin credentials. Gobuster dir then found files and folders that are not linked, including /development.log, /monthly and /private, and vhost mode found three hidden virtual hosts (blog, admin and shop). All scans were run only against the room lab machine.

## 7. Questions I Found Hard
- Why curl -I returned 404 while curl -v returned 200: a HEAD request is handled differently from a GET, so I compared both.
- The difference between a subdomain (DNS) and a virtual host (Host header), which I understood by running the vhost scan.

## 8. Key Learnings (reword in my own words)
1. robots.txt and sitemap.xml are meant for search engines, but they list paths an attacker can read, so sensitive areas must be protected by authentication and not by hiding them.
2. Response headers and framework documentation reveal the software behind a site, and default admin credentials such as admin / admin must be changed.
3. Automated tools such as Gobuster find files and directories nobody links to (a development log, private folders), so exposed log and backup files are a real risk.
4. A subdomain is found through DNS and a virtual host through the Host header, so both need to be checked, and these scans must only be run with permission.
