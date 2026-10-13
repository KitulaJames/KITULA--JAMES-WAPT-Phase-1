# Room 2: Active Recon

**Platform:** TryHackMe
**Room:** https://tryhackme.com/room/activerecon
**Date completed:** 13 October 2026

## 1. Objective
Learn to gather information by interacting directly with a target, using a web browser with Developer Tools, ping, traceroute, telnet and netcat. Active recon sends packets to the target, so it can be logged and detected, and it must only be done with written authorisation. In this room all probing was done against the room's own lab machines.

## 2. Concepts

### 2.1 Active vs passive recon
| | Passive | Active |
|---|---|---|
| Traffic to target | None, public sources only | Yes: packets, connections, probes |
| Leaves traces | No | Yes: access logs, firewall logs, WAF events, IDS alerts |
| Examples | WHOIS, DNS, Shodan, crt.sh | Browsing, ping, traceroute, telnet, nc |
| Rule | Low risk | Never without explicit, signed authorisation |

### 2.2 Browser and Developer Tools
A browser looks like normal traffic, so it is the least suspicious recon tool. Shortcut: Ctrl+Shift+I.

| DevTools tab | What it shows for recon |
|---|---|
| Network | Requests and responses, headers such as Server, X-Powered-By, Content-Security-Policy, status codes, cookies, protocol (h3 = HTTP/3 over QUIC) |
| Console | Run JavaScript in the page |
| Sources | JavaScript, CSS and HTML files; may contain hardcoded API endpoints, internal paths and developer comments |
| Application / Storage | Cookies, Local Storage, Session Storage (tokens, keys exposed client-side) |
| Security | Certificate issuer, validity and Subject Alternative Names (may reveal more subdomains) |

- A non-standard port can be reached by adding it to the URL, e.g. `https://target:8443/`.
- Useful extensions: FoxyProxy (proxy switching), User-Agent Switcher, Wappalyzer (technology fingerprinting).

### 2.3 ping (ICMP)
- Sends an ICMP Echo Request (type 8) and waits for an Echo Reply (type 0). Tells me if a host is reachable.
- Usage: `ping -c 5 IP` (Linux/macOS), `ping -n 5 IP` (Windows). `-4` / `-6` force the IP version. `-s` sets the size of the data in the echo request.
- TTL: the maximum number of routers a packet may pass; each router subtracts 1. Linux usually starts at 64, Windows at 128, so a TTL of 58 probably means Linux about 6 hops away.
- No reply can mean: host down, ICMP blocked by a firewall (Windows Firewall blocks ping by default), NAT, or cloud/CDN filtering.

| Result | Likely meaning | Next step |
|---|---|---|
| Fast replies | Host up, ICMP allowed | Port scan |
| Destination Host Unreachable | Host down or no route | Check the machine |
| 100% loss, no error | ICMP filtered | Try TCP/UDP discovery with Nmap |

### 2.4 traceroute
- Maps the routers (hops) between me and the target by sending packets with TTL 1, 2, 3 and so on. Each router that drops one answers with ICMP "Time Exceeded", revealing its IP.
- Linux: `traceroute IP` (UDP by default), `-T` for TCP, `-I` for ICMP. Windows: `tracert`. `mtr IP` gives a live view with loss and latency per hop.
- Three probes are sent per hop, so I may see up to three IPs and times per line. `*` means no reply (filtered or rate-limited).
- Routes change between runs because of load balancing, anycast and CDNs (the room's two runs to tryhackme.com took 14 and 26 hops).

### 2.5 telnet and netcat (banner grabbing)
- Telnet (port 23) sends everything in cleartext, including passwords; SSH is the secure replacement. But the telnet client can connect to any TCP port, so it is useful for banner grabbing.
- Banner grabbing: connect, read the service's greeting, and optionally send a protocol command. For HTTP: `GET / HTTP/1.1` then `host: example` and two Enters. The `Server:` header reveals the software and version, which can be checked against CVE databases.
- FTP (port 21) and SMTP (25) send a banner immediately on connection.
- Netcat: `nc IP PORT` as a client. As a server: `nc -vnlp 1234` (`-l` listen, `-v` verbose, `-n` no DNS, `-p` port, `-k` keep listening). Ports below 1024 need root.
- Encrypted services: use `curl -I https://IP` or `openssl s_client -connect IP:443`, because telnet and plain nc cannot do TLS.

---

## 3. Practical Work
All commands were run from my Kali Linux machine, connected to the TryHackMe network over OpenVPN, against the room's lab machines. The lab IP addresses change each time a VM is started.

| # | Task | Command | What I found |
|---|---|---|---|
| 1 | Browser DevTools | Developer Tools on the room's website | Total number of questions: 8 |
| 2 | ping | `ping -c 10 TARGET_IP` | 10 replies out of 10, 0% packet loss, TTL 62 (Linux starts at 64, so a Linux host about 2 hops away), average round trip about 308 ms (high because the traffic goes over the VPN) |
| 3 | traceroute | `traceroute TARGET_IP` | 25 hops between my machine and the target |
| 4 | telnet | `telnet TARGET_IP 80`, then `GET / HTTP/1.1` and `host: telnet` | Task 5 lab VM: Apache httpd, version 2.4.61. Task 6 lab VM (extra check): nginx 1.22.1 on the Debian default page |
| 5 | netcat | `nc TARGET_IP 21` | FTP banner: "220 ... FTP server (Version 6.4/OpenBSD/Linux-ftpd-0.17) ready." Version 6.4/OpenBSD/Linux-ftpd-0.17. The banner also reveals the internal hostname (ip-10-49-160-241.ap-south-1.compute.internal) and that the host runs in an AWS region (ap-south-1), without any login |

Traceroute questions from the room text (read the two example outputs):
- Traceroute A, last router before tryhackme.com: 100.92.9.83
- Traceroute B, last router before tryhackme.com: 99.83.89.19
- Traceroute B, number of routers between the two systems: 25 (hops 1 to 25 are routers, and hop 26 is the destination itself)

Ping questions: option to set the size of the data in an echo request = -s; size of the ICMP header in bytes = 8; does Windows Firewall block ping by default = Y (yes, it blocks ping by default on most versions)

**What the banner told me (own words):** the Server header names the web server software and its exact version, so an attacker can look that version up in vulnerability databases. On the Task 6 lab machine even a 400 Bad Request page showed Server: nginx/1.22.1, so error pages leak the version too. Hiding it needs a setting such as server_tokens off in nginx.

### 3.1 Extra practice (own work, safe targets only)
- `ping -c 4 127.0.0.1` on my own Kali machine: 4 sent, 4 received, 0% packet loss, average about 0.08 ms, TTL 64 (the Linux default starting TTL, as the room explains)
- `telnet 10.49.160.241 80` on the Task 6 lab machine: my first attempt sent the invalid text "id", and the server answered 400 Bad Request but still revealed Server: nginx/1.22.1. A valid GET request then returned the normal headers
- `curl -I http://TARGET_IP` on the lab machine: Server: nginx/1.22.1, Content-Type: text/html, Content-Length: 867, Last-Modified: Fri, 08 Oct 2021, ETag and Connection: keep-alive. It gave the same version banner as telnet, in a shorter and cleaner way
- `nc -vnlp 1234` listening on my Kali machine and connecting from a second terminal with `nc 127.0.0.1 1234`: typed text appeared on both sides = yes. Text I typed in the client showed up in the listener and the reply showed up in the client, which is how netcat can act as a simple chat or file channel
- `ping -c 4 TARGET_IP` and compared the TTL with the Linux/Windows guide in Section 2.3: TTL 62 on the lab machine against TTL 64 on my own machine, so about 2 routers in between, and both are consistent with Linux

## 4. Tools Used
| Tool | Use |
|---|---|
| Browser + DevTools | Inspect requests, headers, sources, storage and certificates |
| ping | Check reachability, read TTL |
| traceroute | Map the path to the target |
| telnet | Banner grabbing on TCP ports |
| netcat (nc) | Banner grabbing and a simple listener |
| TryHackMe AttackBox | Machine the commands were run from |

## 5. Evidence
Screenshots are in the `ScreenShots` folder and listed in this folder's README.

## 6. Result
From my Kali machine over the VPN, the lab machine answered all 10 pings with TTL 62, which points to a Linux host about two routers away. Connecting to port 80 with telnet and curl revealed Server: nginx/1.22.1 on a Debian default page, so the exact software version could be read without any login. This shows how simple active recon exposes version information, and also that these probes send traffic to the target, so they must only be used with permission.

## 7. Questions I Found Hard
-

## 8. Key Learnings (reword in my own words)
1. Active recon touches the target, so it leaves logs and can trigger alerts, and it is only legal with explicit written authorisation. Passive recon does not touch the target.
2. A service banner (for example `Server: nginx/1.x`) gives the software and version, which an attacker can look up in vulnerability databases. Defenders should hide or limit banners.
3. ping and its TTL show whether a host is up and hint at its operating system, while traceroute shows the path and where filtering happens. Both can be blocked or give changing results.
4. telnet is insecure because it sends passwords in cleartext, but it and netcat are still useful for talking to any TCP port. For encrypted services, curl or openssl is the right tool.
