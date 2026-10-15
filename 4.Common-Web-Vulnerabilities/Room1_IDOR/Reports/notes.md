# IDOR - Santa's Little IDOR

**Platform:** TryHackMe (Advent of Cyber 2025, Day 5)
**Room:** https://tryhackme.com/room/idor-aoc2025-zl6MywQid9
**Date completed:** 15 October 2026

## 1. Objective
Learn what Insecure Direct Object Reference (IDOR) is, why it happens, and how to find and exploit it on the "TryPresentMe" web application, then understand how to fix it.

## 2. Concepts

### 2.1 What IDOR is
IDOR = Insecure Direct Object Reference. A user-controllable reference (an ID in a URL, a request parameter, a cookie) is used to fetch data, and the server does not check whether the requester is actually allowed to see that specific record. Example query: `SELECT person, address, status FROM Packages WHERE packageID = value;` — if packageID is just a sequential number and ownership is never checked, anyone can view any package by guessing or iterating the ID.

### 2.2 Authentication vs authorization
| Term | Meaning |
|---|---|
| Authentication | Verifying who I am (username/password, then session cookie/token on every later request) |
| Authorization | Verifying what I'm allowed to do once I'm known (can I view this page, make this payment, see this record) |

Authorization can't happen before authentication, since the app must first know who is asking before it can decide what they're allowed to see. If a vulnerable feature needs no login at all, authentication has to be fixed before authorization can be fixed.

### 2.3 Privilege escalation types
| Type | Meaning |
|---|---|
| Vertical | Gaining access to more features (normal user doing admin actions) |
| Horizontal | Using a feature I'm allowed to use, but reaching data I'm not allowed to see (another user's record) |

IDOR is usually horizontal privilege escalation: I'm authorised to use "track my package" or "view my account", but not restricted to only my own data.

### 2.4 Where IDOR hides
| Form | Example | How to exploit |
|---|---|---|
| Plain sequential ID | `user_id=10` | Change the number directly |
| Encoded ID | Base64, e.g. `Mg==` decodes to `2` | Decode, change the number, re-encode |
| Hashed ID | An MD5/SHA1-looking string | Identify the hash algorithm and the input it hashes, then regenerate it for another target value |
| Algorithmically generated ID | A UUID (e.g. version 1) | Understand the generation algorithm; UUIDv1 embeds a timestamp, so if the generation time window is known, every possible UUID in that window can be brute-forced |

### 2.5 Finding IDOR with browser DevTools
1. Log in, open DevTools (right-click > Inspect), go to the Network tab, and reload the page.
2. Inspect the request that loads account info (e.g. `view_accountinfo`) and note the parameter used, e.g. `user_id=10`, and confirm the response matches my own account.
3. Go to the Storage/Application tab, open Local Storage, find the stored auth/user object (e.g. `auth_user`), and edit the `user_id` value.
4. Refresh the page: the app now shows a different user's data, since the server trusted the client-supplied ID without checking ownership.

### 2.6 Fixing IDOR (defender's view)
- The server must check, on every request, whether the authenticated user actually owns or is permitted to access the requested record, not just that a valid session exists.
- Hiding IDs with Base64 or a hash is not a fix; both can be decoded or reproduced if the generation method is known.
- Random/hard-to-guess IDs for public links help, but are not sufficient alone.
- Always test by trying to access another user's data directly, and log and monitor repeated failed access attempts as a sign of an attack in progress.

---

## 3. Practical Work
Target: TryPresentMe web application (lab machine), logged in as `niels` / `TryHackMe#2025`.

| # | Task | What I did | Result |
|---|---|---|---|
| 1 | Basic IDOR via Local Storage | Logged in, inspected the `view_accountinfo` request in DevTools Network tab, found `user_id=10` matched my own account, then edited the `user_id` value in Local Storage (auth_user) to 11 and refreshed | Saw another user's account data, confirming the server trusts the client-supplied ID |
| 2 | Encoded reference | Clicked the "view child" icon, inspected the request, saw a Base64 value (e.g. `Mg==`) | Decoded it to the plain numeric ID (2), confirming the same IDOR exists behind encoding |
| 3 | Hashed reference | Clicked the "edit child" icon, inspected the request, saw an MD5-looking hash | Identified it as a hash of a predictable value, showing the same flaw can hide behind hashing |
| 4 | Algorithmic reference (vouchers) | Decoded a voucher with a UUID decoder tool | Found it was UUIDv1, which embeds a timestamp, so vouchers from a known time window could be brute-forced |
| 5 | Exploit for the flag | Changed `user_id` values via Local Storage until I found the parent account with 10 children | user_id: ____ (niels' own account, user_id likely 10, has 5 children: Bilbo, johny x2, Terst, Test2 — not the target; kept iterating to find the account with exactly 10 children) |

### 3.1 Bonus tasks (optional, own progress)
- Base64/MD5 child endpoint: found the id_number of the child born on 2019-04-17 using Burp Intruder to iterate: ____________
- Voucher claim endpoint: found the voucher valid on 20 Nov 2025 by generating UUIDv1 candidates for 20:00-24:00 UTC: ____________

## 4. Tools Used
Browser DevTools (Network, Storage/Local Storage tabs), a Base64 decoder, a hash identifier, a UUID decoder, Burp Suite Intruder (bonus), TryHackMe AttackBox.

## 5. Evidence
Screenshots are in the `ScreenShots` folder and listed in this folder's README.

## 6. Result
____________ (2 to 3 sentences: what changing the user_id revealed, and what the encoded/hashed/UUID examples showed about how IDOR can be disguised but still exploited.)

## 7. Questions I Found Hard
-

## 8. Key Learnings (own words)
1. IDOR happens when the server uses a client-supplied reference to fetch data without checking that the requester actually owns it, so a simple ID change can expose another user's data.
2. Authorization must happen on every single request, not just at login, since a valid session only proves who I am, not what I'm allowed to see.
3. Encoding (Base64) and hashing an ID are not security controls; both can be reversed or reproduced once the underlying value or algorithm is known.
4. IDOR is usually horizontal privilege escalation: I keep my normal access level but reach data belonging to someone else, which is why ownership checks matter more than hiding the ID.
