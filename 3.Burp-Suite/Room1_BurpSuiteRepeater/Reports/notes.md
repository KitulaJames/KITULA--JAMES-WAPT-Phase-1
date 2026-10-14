# Burp Suite: Repeater

**Platform:** TryHackMe
**Room:** https://tryhackme.com/room/burpsuiterepeater
**Date completed:** 15 October 2026

## 1. Objective
Learn to use Burp Suite's Repeater module to capture, edit and resend HTTP requests, and use it to test an endpoint for a SQL injection vulnerability.

## 2. Concepts

### 2.1 What Repeater is for
Repeater lets me take a request (usually captured in Proxy) and resend it with changes, as many times as I want. It is built for manual, repetitive testing: SQLi testing, bypassing filters, and tweaking form parameters. I can also build a request from scratch, like a GUI version of curl.

### 2.2 Repeater interface
| Section | What it does |
|---|---|
| Request list | Top left; lists every request sent to Repeater, each kept separately |
| Request controls | Send, cancel, and step back/forward through history |
| Request and Response view | The main area; edit the request on one side, see the response on the other |
| Layout options | Side-by-side, vertical, or separate tabs |
| Inspector | A structured, clickable view of the request/response (more intuitive than raw text) |
| Target | Shows the host the request goes to; fills in automatically when sent from Proxy |

### 2.3 Workflow
Capture a request in Proxy, then right-click > Send to Repeater, or Ctrl+R. The Request view fills with the captured request; the Response view is empty until I click Send. Editing the request text and clicking Send again updates the Response view. The history buttons next to Send let me step back and forward through previous versions of the request.

### 2.4 Response view options
| View | Shows |
|---|---|
| Pretty (default) | Lightly formatted for readability |
| Raw | Exactly what the server sent, no formatting |
| Hex | Byte-level view, useful for binary data |
| Render | The page as a browser would draw it |

The "show non-printable characters" button (\n) reveals characters like the \r\n line endings, which matter for how HTTP headers are parsed.

### 2.5 Inspector
A structured breakdown of a request/response, usable in both Proxy and Repeater. Request sections are editable; response headers are view-only (I don't control what the server sends back).

| Section | What it holds |
|---|---|
| Request Attributes | Method, path, HTTP version |
| Request Query Parameters | Data in the URL after ?, e.g. ?redirect=false |
| Request Body Parameters | Data sent in a POST body |
| Request Cookies | Cookies sent with the request |
| Request Headers | All request headers, editable |
| Response Headers | Headers the server returned (view only) |

### 2.6 Testing with Repeater
Two simple exercises: adding a custom header (e.g. FlagAuthorised: True) to see how the server reacts, and sending extreme values to a numeric endpoint (e.g. /products/<huge or invalid number>) to try to trigger a 500 Internal Server Error, which can reveal how the backend handles bad input.

### 2.7 Extra-mile challenge: Union SQL injection
Target: the ID parameter of /about/ID.

1. Capture `GET /about/2` in Proxy, send to Repeater.
2. Add a single apostrophe after the ID (`/about/2'`) and send. A `500 Internal Server Error` confirms the input breaks the SQL query.
3. The verbose error message leaked the real query: `SELECT firstName, lastName, pfpLink, role, bio FROM people WHERE id = 2'`. This revealed the table name (`people`) and 5 selected columns, skipping manual enumeration of both.
4. Used an ID that doesn't exist (0) with a UNION SELECT, so only the injected row returns: `/about/0 UNION ALL SELECT column_name,null,null,null,null FROM information_schema.columns WHERE table_name="people"`. The 4 nulls pad the column count to match the original query (a UNION needs equal columns on both sides).
5. Used `group_concat(column_name)` to pull every column name in one response instead of one at a time: found id, firstName, lastName, pfpLink, role, shortRole, bio, notes.
6. The CEO's user ID (1) was found by opening his profile from the /about/ page and reading the ID in the URL.
7. Final query: `0 UNION ALL SELECT notes,null,null,null,null FROM people WHERE id = 1`, which returned the CEO's private notes as the flag.

**Why this worked:** the app concatenated user input directly into a SQL query with no sanitisation, and its error page echoed the real SQL statement, giving an attacker everything needed to escalate from a single `'` to reading any column of any table.

---

## 3. Practical Work
| # | Task | What I did | Result |
|---|---|---|---|
| 1 | Repeater basics | Captured `GET /` in Proxy, sent to Repeater, viewed the response in Pretty/Raw/Hex | Confirmed Inspector gives more intuitive control than editing raw text |
| 2 | Custom header | Added `FlagAuthorised: True` as a request header and resent | Flag: THM{Yzg2MWI2ZDhlYzdlNGFiZTUzZTIzMzVi} |
| 3 | 500 error | Sent extreme values to a numeric /products/ID endpoint until the server returned 500 | Flag: THM{N2MzMzFhMTA1MmZiYjA2YWQ4M2ZmMzhl} |
| 4 | Union SQL injection | Followed the steps in Section 2.7 against /about/ID | CEO's notes (flag): THM{ZGE3OTUyZGMyMzkwNjJmZjg3Mzk1NjJh} |

**What I observed (own words):** Repeater made it quick to repeat the same request with small changes (one header, one number, one SQL payload) and compare responses side by side. The SQL injection showed how a single overly verbose error message gave away the table name and column count, turning a basic apostrophe test into full data extraction with a UNION query.

## 4. Tools Used
Burp Suite Community (Proxy, Repeater, Inspector), Firefox (Burp's built-in browser), TryHackMe AttackBox / Kali.

## 5. Evidence
Screenshots are in the `ScreenShots` folder and listed in this folder's README.

## 6. Result
Repeater let me capture a request once and then resend it repeatedly with small edits, which made manual testing of headers, numeric parameters and SQL payloads fast and precise. The /about/ID endpoint was vulnerable to a Union-based SQL injection: a single apostrophe broke the query and the server's verbose error leaked the table name and column list, which was enough to pull the CEO's private notes from the database without any authentication.

## 7. Questions I Found Hard
-

## 8. Key Learnings (own words)
1. Repeater turns a captured request into something I can edit and resend as many times as needed, which makes manual testing practical.
2. A verbose error message that echoes the actual SQL query is a serious information leak on its own, since it hands an attacker the table and column names.
3. A UNION-based SQL injection needs the same number of columns as the original query, so padding with NULLs (or finding the real column count) is a key step.
4. Directly concatenating user input into a database query, with no parameterisation or input validation, is what makes this class of vulnerability possible.
