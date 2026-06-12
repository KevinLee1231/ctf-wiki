# CTF Misc - Oracles, Recurrences, CAPTCHA and Polyglots

Additional CTF-era challenges extracted from 2018+ writeups. For earlier parts, see vm-z3-sandbox-and-game-basics.md, game-state-websocket-and-wasm.md, and interactive-containers-jails-and-solvers.md.

## 阅读定位

- 本卷是 **后续补卷**，更偏 oracle、递推、验证码、多层 polyglot 和年代补录案例。
- 如果还没有先完成题型分流，不要从这里起步。


## Table of Contents
- [XSLT as Turing-Complete VM for Binary Search (35C3 2018)](#xslt-as-turing-complete-vm-for-binary-search-35c3-2018)
- [JavaScript MAX_SAFE_INTEGER Successor Equality (35C3 2018)](#javascript-max_safe_integer-successor-equality-35c3-2018)
- [Binary Search Oracle in Comparison-Only DSL (35C3 2018)](#binary-search-oracle-in-comparison-only-dsl-35c3-2018)
- [Blind SQLi via Script-Engine Timeout Error (35C3 2018)](#blind-sqli-via-script-engine-timeout-error-35c3-2018)
- [OEIS Sequence Lookup Automation for Recurrence Puzzles (X-MAS CTF 2018)](#oeis-sequence-lookup-automation-for-recurrence-puzzles-x-mas-ctf-2018)
- [QR Code Reassembly from Format-String Structural Constraints (Square CTF 2018)](#qr-code-reassembly-from-format-string-structural-constraints-square-ctf-2018)
- [Matrix Exponentiation for Fibonacci-Like Recurrence (Pwn2Win 2018)](#matrix-exponentiation-for-fibonacci-like-recurrence-pwn2win-2018)
- [Tribonacci Recurrence for Frog Jump Counting (FireShell 2019)](#tribonacci-recurrence-for-frog-jump-counting-fireshell-2019)
- [Selenium + Tesseract for Dynamic Font CAPTCHA (Square CTF 2018)](#selenium--tesseract-for-dynamic-font-captcha-square-ctf-2018)
- [Brainfuck Decodes Piet Image URL — Multi-Layer Polyglot (RITSEC 2018)](#brainfuck-decodes-piet-image-url--multi-layer-polyglot-ritsec-2018)
- [Bytebeat Synth Code Recognition for Hidden Audio (RITSEC 2018)](#bytebeat-synth-code-recognition-for-hidden-audio-ritsec-2018)

---

## XSLT as Turing-Complete VM for Binary Search (35C3 2018)

```xml
<xsl:template name="step">
  <xsl:param name="lo"/><xsl:param name="hi"/>
  <xsl:variable name="mid" select="($lo + $hi) div 2"/>
  <xsl:choose>
    <xsl:when test="$target = $mid">...found...</xsl:when>
    <xsl:when test="$target &lt; $mid">
      <xsl:call-template name="step">
        <xsl:with-param name="lo" select="$lo"/>
        <xsl:with-param name="hi" select="$mid"/>
      </xsl:call-template>
    </xsl:when>
    ...
  </xsl:choose>
</xsl:template>
```

**关键结论：** Any "pure template" language with named recursion and conditionals is a VM. Build a primitive (binary search, bit extraction, state accumulator) out of its native constructs before trying to escape the sandbox.

**参考：** 35C3 CTF 2018 — Juggle, writeup 12803

---

## JavaScript MAX_SAFE_INTEGER Successor Equality (35C3 2018)

```js
let x = 9007199254740992; // 2^53
console.log(x === x + 1); // true
```

**关键结论：** Any numeric invariant that compares `n` to `n + 1` fails at the float boundary. Test with `2^53`, `Infinity`, `NaN`, and `-0 === 0` combinations when a JS check looks like it's making assumptions about arithmetic.

**参考：** 35C3 CTF 2018 — Number Error, writeup 12828

---

## Binary Search Oracle in Comparison-Only DSL (35C3 2018)

```python
guess = 0
for shift in range(30, -1, -1):
    guess += 1 << shift
    if oracle(guess) > 0:     # guess too high
        guess -= 1 << shift
```

**关键结论：** Any boolean comparator gives you binary search in `O(log N)` queries. The same trick collapses any comparison-based leak — regex match, timing channel, HTTP status code — into the full value.

**参考：** 35C3 CTF 2018 — Juggle, writeup 12803

---

## Blind SQLi via Script-Engine Timeout Error (35C3 2018)

```lua
-- blind timing oracle in Lua eval sandbox
for c in printable do
    send(("if charAt(FLAG, %d) == '%s' then pause(10000) end"):format(i, c))
    if response_time > 5 then flag = flag .. c; break end
end
```

**关键结论：** Script-eval services with timeouts are stateful oracles: any long-running expression leaks a boolean via the wall-clock difference between timeout and instant return.

**参考：** 35C3 CTF 2018 — dev/null, writeups 12830, 12871

---

## OEIS Sequence Lookup Automation for Recurrence Puzzles (X-MAS CTF 2018)

```python
import requests
from pyquery import PyQuery as pq
r = requests.get('https://oeis.org/search', params={'q': ','.join(map(str, seq))})
doc = pq(r.text)
next_term = doc('pre').eq(1).text().split(',')[len(seq)]
```

**关键结论：** Any integer-sequence puzzle is solved in one HTTP request via OEIS. The hard part is the wrapper (captcha, PoW, socket framing) — automate that once and the math stops being the bottleneck.

**参考：** X-MAS CTF 2018 — A Weird List of Sequences, writeup 12683

---

## QR Code Reassembly from Format-String Structural Constraints (Square CTF 2018)

```python
wanted_formats = load_32_valid_qr_formats()
for col in slices:
    if col[:7] in wanted_formats_column_8:
        candidate_cols.append(col)
for perm in itertools.permutations(candidate_cols):
    if decode_qr(np.stack(perm)):
        return perm
```

**关键结论：** Format-specific constraints collapse permutation spaces. QR Version 1 has only 32 possible format strings; anchor on them to prune before brute-forcing.

**参考：** Square CTF 2018 — C3: Shredded, writeup 12331

---

## Matrix Exponentiation for Fibonacci-Like Recurrence (Pwn2Win 2018)

```python
MOD = 10**9 + 7
def matmult(a, b):
    return ((a[0]*b[0] + a[1]*b[2]) % MOD, (a[0]*b[1] + a[1]*b[3]) % MOD,
            (a[2]*b[0] + a[3]*b[2]) % MOD, (a[2]*b[1] + a[3]*b[3]) % MOD)
def matpow(M, n):
    R = (1,0,0,1)
    while n:
        if n & 1: R = matmult(R, M)
        M = matmult(M, M); n >>= 1
    return R
```

**关键结论：** Any linear recurrence over a ring is reducible to matrix exponentiation. Use it whenever the challenge exposes a giant `N` for a classical-looking sequence — Fibonacci, Tribonacci, Lucas, linear Pisano, RNG counters.

**参考：** Pwn2Win CTF 2018 — Too Slow, writeup 12501

---

## Tribonacci Recurrence for Frog Jump Counting (FireShell 2019)

```python
def tribonacci(N, MOD=13371337):
    a, b, c = 0, 0, 1
    for _ in range(N):
        a, b, c = b, c, (a + b + c) % MOD
    return c
```

**关键结论：** "Number of ways to climb N stairs with step sizes {1..k}" is always a linear recurrence. Memoize up to the server's max `N`, cache across requests, and keep the tribonacci identity in mind when the challenge text mentions "frog".

**参考：** FireShell CTF 2019 — Frogs, writeup 12961

---

## Selenium + Tesseract for Dynamic Font CAPTCHA (Square CTF 2018)

```python
from selenium import webdriver
from PIL import Image
import pytesseract, io
d = webdriver.Chrome()
d.get(URL); d.execute_script("document.body.style.zoom='450%'")
img = Image.open(io.BytesIO(d.get_screenshot_as_png()))
expr = pytesseract.image_to_string(img).replace('x','*').replace('{','(').replace('}',')')
d.execute_script(f"document.getElementsByName('answer')[0].value={eval(expr)}")
d.find_element_by_tag_name('form').submit()
```

**关键结论：** Dynamic CAPTCHAs are often too short-lived for manual solves but trivial for a 1-second Selenium + Tesseract loop. When OCR alone fails, pair it with a cmap reference library (see ctf-osint/web-and-dns.md).

**参考：** Square CTF 2018 — C8, writeups 12160, 12178

---

## Brainfuck Decodes Piet Image URL — Multi-Layer Polyglot (RITSEC 2018)

```bash
bf puzzle.bf                          # prints youtube.com/watch?v=XXXX
yt-dlp -x --write-thumbnail "$URL"    # grabs JPG thumbnail
python crop_border.py thumb.jpg > piet.png
npiet piet.png                        # prints the flag
```

**关键结论：** Multi-layer esolangs are recognisable by eye: Brainfuck is `+-<>.,[]`, Piet is colored block grids, Whitespace is invisible. If a challenge description mentions multiple "weird" formats, pipeline the decoders in order.

**参考：** RITSEC CTF 2018 — writeup 12224

---

## Bytebeat Synth Code Recognition for Hidden Audio (RITSEC 2018)

```c
/* Bytebeat example: output byte = low 8 bits of this expression */
(t * ((t >> 12 | t >> 8) & 63 & t >> 4))
```

**关键结论：** Recognise bytebeat by (a) a `t` variable, (b) bitshifts mixed with modulo, (c) output of size 8-bit unsigned integer. `%`, `|`, `&`, `^`, `>>`, `<<` on `t` are the bytebeat signature. No decoding needed — just play it.

**参考：** RITSEC CTF 2018 — writeups 12261, 12268

---


---

## SHA-256 Length Extension Attack

MAC = `SHA-256(SECRET || msg)` with known msg/hash -> forge valid MAC via `hlextend`. Vulnerable: SHA-256, MD5, SHA-1. NOT: HMAC, SHA-3.

```python
import hlextend
sha = hlextend.new('sha256')
new_data = sha.extend(b'extension', b'original_message', len_secret, known_hash_hex)
```


---

## Levenshtein Distance Oracle Attack (SunshineCTF 2016)

Oracle returns edit distance between guess and secret. Determine length from empty string, identify present chars from single-char repeats, binary search for positions. O(n log n) queries. See interactive-containers-jails-and-solvers.md.
