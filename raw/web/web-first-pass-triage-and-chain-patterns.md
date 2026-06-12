# CTF Web - First-Pass Triage and Chain Patterns

## 阅读定位

Web 题首轮问题、侦察检查、常见漏洞链和 flag/artifact 搜索位置。


## First-Pass Workflow

1. Identify the real boundary: browser only, backend only, mixed app, or auth flow.
2. Capture one normal request/response pair for every major feature before fuzzing.
3. Enumerate hidden functionality from JS bundles, response headers, routes, and alternate methods.
4. Classify the likely bug family: injection, authz, parser mismatch, upload, trust proxy, state machine, or client-side execution.
5. Build the smallest proof first: leak, bypass, or primitive. Save full exploit chaining for later.


---

## First Questions to Answer

- Is the flag likely in the browser, an API response, a local file, a database row, or an internal service?
- Does the app trust user-controlled data in templates, redirects, file paths, headers, serialized objects, or background jobs?
- Are there multiple parsers disagreeing with each other: proxy vs app, URL parser vs fetcher, sanitizer vs browser, serializer vs filter?
- Can you turn the bug into a smaller primitive first: read one file, forge one token, call one internal endpoint, trigger one bot visit?


---

## High-Value Recon Checks

- Read the HTML, inline scripts, and bundled JS before guessing the API surface.
- Compare what the UI submits with what the backend accepts; optional JSON fields often unlock hidden paths.
- Check obvious metadata and helper paths early: `/robots.txt`, `/sitemap.xml`, `/.well-known/`, `/admin`, `/debug`, `/.git/`, `/.env`.
- Try alternate verbs and content types on interesting routes: `GET`, `POST`, `PUT`, `PATCH`, `TRACE`, JSON, form, multipart, XML.
- Treat file upload, PDF/export, webhook, OAuth callback, and admin bot features as likely exploit multipliers.


---

## Fast Pattern Map

- SQL errors, odd filtering, or state-dependent DB behavior: start with sql-injection.md.
- Templating, file reads, SSRF, command execution, XML, or parser bugs: start with php-lfi-ssti-ssrf-and-type-juggling.md and ruby-php-upload-and-ssti-rce.md.
- XSS, CSP bypass, admin bot, client routing, DOM issues, or scriptless exfiltration: start with xss-dom-and-browser-tricks.md.
- Session forgery, hidden admin routes, JWT, OAuth, SAML, or weak trust boundaries: start with auth-bypass-cookies-and-hidden-routes.md, auth-jwt.md, and oauth-saml-cors-and-cicd.md.
- Node.js apps, prototype pollution, VM sandboxes, or SSRF into internal services: add node-and-prototype.md.
- Smart contract frontends or blockchain-integrated apps: add web3.md.


---

## Common Chain Shapes

- Recon -> hidden route -> auth bypass -> internal file read -> token or flag
- XSS or HTML injection -> admin bot -> privileged action -> secret leak
- Traversal or upload -> config/source leak -> secret recovery -> session forgery
- SSRF -> metadata or internal API -> credential leak -> code execution
- SQLi or NoSQL injection -> credential bypass -> second-stage template or upload abuse


---

## Web 首轮深挖转向条件
Use the specific Web references from index.md once you have confirmed the challenge is truly web-heavy and need a deeper exploit catalog.

- Recon, SQLi, XSS, traversal, JWT, SSTI, SSRF, XXE, and command injection quick notes
- Deserialization, race conditions, file upload to RCE, and multi-stage chain examples
- Node, OAuth/SAML, CI/CD, Web3, bot abuse, CSP bypasses, and modern browser tricks
- CVE-shaped playbooks and older challenge patterns that still show up in modern CTFs


---

## Common Flag Locations

- Files: `/flag.txt`, `/flag`, `/app/flag.txt`, `/home/*/flag*`
- Environment: `/proc/self/environ`, process command line, debug config dumps
- Database: tables named `flag`, `flags`, `secret`, or seeded challenge content
- HTTP: custom headers, archived responses, hidden routes, admin exports
- Browser: hidden DOM nodes, `data-*` attributes, inline state objects, source maps


---

### 速查补充

Files: `/flag.txt`, `/flag`, `/app/flag.txt`, `/home/*/flag*`. Env: `/proc/self/environ`. DB: `flag`, `flags`, `secret` tables. Headers: `x-flag`, `x-archive-tag`, `x-proof`. DOM: `display:none` elements, `data-*` attributes.
## Reconnaissance
- View source for HTML comments, check JS/CSS files for internal APIs
- Look for `.map` source map files
- Check response headers for custom X- headers and auth hints
- Common paths: `/robots.txt`, `/sitemap.xml`, `/.well-known/`, `/admin`, `/api`, `/debug`, `/.git/`, `/.env`
- Search JS bundles: `grep -oE '"/api/[^"]+"'` for hidden endpoints
- Check for client-side validation that can be bypassed
- Compare what the UI sends vs. what the API accepts (read JS bundle for all fields)
- Check assets returning 404 status — `favicon.ico`, `robots.txt` may contain data despite error codes: `strings favicon.ico | grep -i flag`
- Tor hidden services: `feroxbuster -u 'http://target.onion/' -w wordlist.txt --proxy socks5h://127.0.0.1:9050 -t 10 -x .txt,.html,.bak`


---

## Multi-Stage Chain Patterns
**0xClinic chain:** Password inference → path traversal + ReDoS oracle (leak secrets from `/proc/1/environ`) → CRLF injection (CSP bypass + cache poisoning + XSS) → urllib scheme bypass (SSRF) → `.so` write via path traversal → RCE

**Key chaining insights:**
- Path traversal + any file-reading primitive → leak `/proc/*/environ`, `/proc/*/cmdline`
- CRLF in headers → CSP bypass + cache poisoning + XSS in one shot
- Arbitrary file write in Python → `.so` hijacking or `.pyc` overwrite for RCE
- Lowercased response body → use hex escapes (`\x3c` for `<`)


---
