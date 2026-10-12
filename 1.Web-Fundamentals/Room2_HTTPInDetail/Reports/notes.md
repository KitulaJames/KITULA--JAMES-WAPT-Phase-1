# Room 2: HTTP in Detail

**Platform:** TryHackMe
**Room:** https://tryhackme.com/room/httpindetail
**Path:** Pre Security > How The Web Works
**Date completed:** 10/10/2026

---

## 1. Objective
Learn how a browser asks a web server for content using HTTP: how a URL is built, what requests and responses look like, and how methods, status codes, headers and cookies work. Then practise making requests in an emulator.

---

## 2. Concepts

### 2.1 HTTP and HTTPS
- HTTP (HyperText Transfer Protocol) is the set of rules browsers and web servers use to send web pages, images and videos.
- HTTPS is the secure version. Its data is encrypted, so people cannot read what is sent or received, and it gives assurance that I am talking to the real server and not someone impersonating it.
- Challenge in the room: a mock web page had an invalid certificate. Flag: `THM{INVALID_HTTP_CERT}`.

![Task 1](../ScreenShots/01_task1_what_is_http.png)

### 2.2 URL (Uniform Resource Locator)
Example from the room: `http://user:password@tryhackme.com:80/view-room?id=1#task3`

| Part | Example | Meaning |
|---|---|---|
| Scheme | http | Protocol used (HTTP, HTTPS, FTP) |
| User | user:password | Login details in the URL (unsafe) |
| Host | tryhackme.com | Domain name or IP of the server |
| Port | 80 | Usually 80 for HTTP and 443 for HTTPS; can be 1-65535 |
| Path | /view-room | The file or location of the resource |
| Query string | ?id=1 | Extra data for the path, e.g. `/blog?id=1` asks for article 1 |
| Fragment | #task3 | Jumps to a part of the page |

### 2.3 Request
Smallest possible request: `GET / HTTP/1.1`. A fuller request:

    GET / HTTP/1.1
    Host: tryhackme.com
    User-Agent: Mozilla/5.0 Firefox/87.0
    Referer: https://tryhackme.com/

- Line 1: method (GET), path (/), HTTP version (1.1).
- Host: which website I want. User-Agent: my browser. Referer: the page that sent me here.
- A request ends with a blank line.

### 2.4 Response

    HTTP/1.1 200 OK
    Server: nginx/1.15.8
    Date: Fri, 09 Apr 2021 13:34:03 GMT
    Content-Type: text/html
    Content-Length: 98

    <html>...</html>

- Line 1: version, status code, and reason (200 OK means success).
- Server: server software and version (information leak). Date: server time.
- Content-Type: what kind of data is returned. Content-Length: how much data to expect (confirms nothing is missing).
- Blank line, then the body (the page).

### 2.5 HTTP methods
| Method | Use | Example |
|---|---|---|
| GET | Get information | View a news article |
| POST | Submit data, may create a record | Create a new user account |
| PUT | Submit data to update information | Update my email address |
| DELETE | Delete a record | Remove an uploaded picture |

### 2.6 Status codes
| Range | Meaning |
|---|---|
| 100-199 | Information: first part accepted, continue sending |
| 200-299 | Success |
| 300-399 | Redirection |
| 400-499 | Client error (problem with my request) |
| 500-599 | Server error |

| Code | Name | Meaning |
|---|---|---|
| 200 | OK | Request completed |
| 201 | Created | A new resource was created (user, blog post) |
| 301 | Moved Permanently | Page moved for good |
| 302 | Found | Temporary redirect |
| 400 | Bad Request | Something wrong or missing in my request |
| 401 | Not Authorised | I must log in first |
| 403 | Forbidden | Not allowed, even if logged in |
| 404 | Not Found | The resource does not exist |
| 405 | Method Not Allowed | Wrong method for that resource (GET on a POST-only page) |
| 500 | Internal Server Error | Server failed and could not handle it |
| 503 | Service Unavailable | Overloaded or down for maintenance |

### 2.7 Headers
| Request header | Purpose |
|---|---|
| Host | Which website I want on a shared server |
| User-Agent | My browser and version |
| Content-Length | How much data is in the body |
| Accept-Encoding | Compression methods my browser supports |
| Cookie | Data sent back to the server to remember me |

| Response header | Purpose |
|---|---|
| Set-Cookie | Tells the browser to store a cookie |
| Cache-Control | How long the browser may keep the content |
| Content-Type | Type of data returned (HTML, CSS, image, PDF) |
| Content-Encoding | Compression used on the data |

### 2.8 Cookies
- A cookie is a small piece of data stored on my computer. The server sets it with the `Set-Cookie` header, and my browser sends it back on every further request.
- HTTP is stateless (it does not remember earlier requests), so cookies remind the server who I am, my settings, or that I logged in.
- Most often cookies are used for authentication. The value is a token (a secret code that is hard to guess), not a clear-text password.
- To view them: DevTools > Network > click a request > Cookies tab.

---

## 3. Practical Work (request emulator)
Used the "View Site" emulator to make requests.

| # | Request | Parameters | What I observed / Flag |
|---|---|---|---|
| 1 | GET /room | none | THM{YOU'RE_IN_THE_ROOM} |
| 2 | GET /blog | id=1 (URI parameter) | THM{YOU_FOUND_THE_BLOG} |
| 3 | DELETE /user/1 | none | THM{USER_IS_DELETED} |
| 4 | PUT /user/2 | username=admin (body parameter) | THM{USER_HAS_UPDATED} |
| 5 | POST /login | username=thm, password=letmein (body) | THM{HTTP_REQUEST_MASTER} |

**Security thinking (own words):** PUT, DELETE and POST change data. A real server must check that I am logged in and allowed to do it (401 and 403 codes), otherwise anyone could change or delete other users. A login sent over HTTP instead of HTTPS can be read on the network.

![Practical 1](../ScreenShots/08_practical_get_room.png)
![Practical 2](../ScreenShots/09_practical_get_blog.png)
![Practical 3](../ScreenShots/10_practical_delete_user1.png)
![Practical 4](../ScreenShots/11_practical_put_user2.png)
![Practical 5](../ScreenShots/12_practical_post_login.png)

---

## 4. Tools Used
- TryHackMe (room and request emulator)
- Browser developer tools (Network tab)

## 5. Result
Completed all tasks (100%), answered every question, found the invalid certificate flag, and made five requests (GET, GET with a parameter, DELETE, PUT, POST) in the emulator.

![Room completed](../ScreenShots/13_room_completion.jpeg)

## 6. Questions I Found Hard
-

## 7. Key Learnings (own words)
1. A request has a method, a path, a version, headers and sometimes a body. The response has a status code, headers and a body.
2. Method choice matters: GET reads, POST creates, PUT updates and DELETE removes. Wrong methods return 405.
3. Status codes show what happened: 401 means log in first, 403 means not allowed, 404 means missing and 500 means a server fault.
4. HTTP is stateless, so cookies carry my identity to the server on every request. Because a cookie can act as a password, it must be protected and never shared.
