# CTF Crypto - Encodings, QR and Esolangs

## Table of Contents
- [Common Encodings](#common-encodings)
  - [Base64](#base64)
  - [Base32](#base32)
  - [Hex](#hex)
  - [IEEE 754 Floating Point Encoding](#ieee-754-floating-point-encoding)
  - [UTF-16 Endianness Reversal (LACTF 2026)](#utf-16-endianness-reversal-lactf-2026)
  - [BCD (Binary-Coded Decimal) Encoding (VuwCTF 2025)](#bcd-binary-coded-decimal-encoding-vuwctf-2025)
  - [Multi-Layer Encoding Detection (0xFun 2026)](#multi-layer-encoding-detection-0xfun-2026)
  - [URL Encoding](#url-encoding)
  - [ROT13 / Caesar](#rot13--caesar)
  - [Caesar Brute Force](#caesar-brute-force)
- [Structured and Low-Frequency Encodings](#structured-and-low-frequency-encodings)
  - [Gray Code Cyclic Encoding (EHAX 2026)](#gray-code-cyclic-encoding-ehax-2026)
  - [Binary Tree Path Encoding](#binary-tree-path-encoding)
  - [RFC 4042 UTF-9 Decoding (SECCON 2015)](#rfc-4042-utf-9-decoding-seccon-2015)
  - [Pixel Color Binary Encoding (Break In 2016)](#pixel-color-binary-encoding-break-in-2016)
  - [TOPKEK Binary Encoding (Hack The Vote 2016)](#topkek-binary-encoding-hack-the-vote-2016)
  - [DTMF Digits to Multi-Tap Keypad Text (h4ckc0n 2017)](#dtmf-digits-to-multi-tap-keypad-text-h4ckc0n-2017)
- [QR Codes](#qr-codes)
  - [Basic Commands](#basic-commands)
  - [QR Structure](#qr-structure)
  - [Repairing Damaged QR](#repairing-damaged-qr)
  - [Finder Pattern Template](#finder-pattern-template)
  - [QR Code Chunk Reassembly (LACTF 2026)](#qr-code-chunk-reassembly-lactf-2026)
  - [QR Code Chunk Reassembly via Indexed Directories (UTCTF 2026)](#qr-code-chunk-reassembly-via-indexed-directories-utctf-2026)
  - [Hexadecimal Sudoku + QR Distribution (BSidesSF 2026)](#hexadecimal-sudoku--qr-distribution-bsidessf-2026)
  - [MaxiCode 2D Barcode Decoding (CSAW CTF 2016)](#maxicode-2d-barcode-decoding-csaw-ctf-2016)
- [Multi-Stage URL Encoding Chain (UTCTF 2026)](#multi-stage-url-encoding-chain-utctf-2026)
- [Esoteric Languages](#esoteric-languages)
  - [Whitespace Language Parser (BYPASS CTF 2025)](#whitespace-language-parser-bypass-ctf-2025)
  - [Custom Brainfuck Variants (Themed Esolangs)](#custom-brainfuck-variants-themed-esolangs)
  - [Multi-Layer Esoteric Language Chains (Break In 2016)](#multi-layer-esoteric-language-chains-break-in-2016)
    - [Brainfuck to Piet Thumbnail Pipeline (RITSEC 2018)](#brainfuck-to-piet-thumbnail-pipeline-ritsec-2018)

---

## Common Encodings

**首选工具 CyberChef** — 遇到不认识编码先拖进 CyberChef，Magic 按钮自动识别。本地：`D:\CTF工具\CyberChef\CyberChef_v10.23.0.html`，在线：https://gchq.github.io/CyberChef/。

### Base64
```bash
echo "encoded" | base64 -d
# Charset: A-Za-z0-9+/=
```

### Base32
```bash
echo "OBUWG32DKRDHWMLUL53TI43OG5PWQNDSMRPXK3TSGR3DG3BRNY4V65DIGNPW2MDCGFWDGX3DGBSDG7I=" | base32 -d
# Charset: A-Z2-7= (no lowercase, no 0,1,8,9)
```

### Hex
```bash
echo "68656c6c6f" | xxd -r -p
```

### IEEE 754 Floating Point Encoding

Numbers that encode ASCII text when viewed as raw IEEE 754 bytes:

```python
import struct

values = [240600592, 212.2753143310547, 2.7884192016691608e+23]

# Each float32 packs to 4 ASCII bytes
for v in values:
    packed = struct.pack('>f', v)  # Big-endian single precision
    print(f"{v} -> {packed}")      # b'Meta', b'CTF{', b'fl04'

# For double precision (8 bytes per value):
# struct.pack('>d', v)
```

**关键结论：** If challenge gives a list of numbers (mix of integers, decimals, scientific notation), try packing each as IEEE 754 float32 (`struct.pack('>f', v)`) — the 4 bytes often spell ASCII text.

### UTF-16 Endianness Reversal (LACTF 2026)

**模式（endians）：** Text "turned to Japanese" -- mojibake from UTF-16 endianness mismatch.

**Fix:** Reverse the encoding/decoding order:
```python
# If encoded as UTF-16-LE but decoded as UTF-16-BE:
fixed = mojibake.encode('utf-16-be').decode('utf-16-le')

# If encoded as UTF-16-BE but decoded as UTF-16-LE:
fixed = mojibake.encode('utf-16-le').decode('utf-16-be')
```

### BCD (Binary-Coded Decimal) Encoding (VuwCTF 2025)

```python
def bcd_decode(data):
    """Decode BCD: each byte = 2 decimal digits."""
    return ''.join(f'{(b>>4)&0xf}{b&0xf}' for b in data)

# Then convert decimal string to ASCII
ascii_text = ''.join(chr(int(decoded[i:i+2])) for i in range(0, len(decoded), 2))
```

### Multi-Layer Encoding Detection (0xFun 2026)

**模式（139 steps）：** Recursive decoding with troll flags as decoys.

**Critical rule:** When data is all hex chars (`0-9a-f`) and has even length, decode as **hex first**, not Base64. Base64, Base32 and text codecs can accept inputs that were not intended for them, so “the first decoder that does not throw” is not evidence of a correct layer.

```python
import base64
import bz2
import codecs
import re
import string
import zlib

MAGICS = (b'PK\x03\x04', b'\x89PNG\r\n\x1a\n', b'%PDF-', b'\x7fELF', b'BZh')

def quality(blob):
    """Score a candidate instead of accepting the first non-exception."""
    if isinstance(blob, str):
        blob = blob.encode('utf-8', errors='replace')
    if not blob:
        return -1.0
    printable = sum(chr(b) in string.printable for b in blob) / len(blob)
    marker = 0.35 if any(x in blob.lower() for x in (b'flag', b'ctf{', b'http')) else 0
    magic = 0.50 if blob.startswith(MAGICS) else 0
    return printable + marker + magic

def candidates(data):
    raw = data.encode() if isinstance(data, str) else data
    text = raw.decode('ascii', errors='ignore').strip()
    out = []

    # Apply format-specific guards before decoding.
    if re.fullmatch(r'[0-9a-fA-F]+', text or 'x') and len(text) % 2 == 0:
        out.append(('hex', bytes.fromhex(text)))
    if re.fullmatch(r'[01]+', text or 'x') and len(text) % 8 == 0:
        out.append(('binary', bytes(int(text[i:i+8], 2) for i in range(0, len(text), 8))))
    if re.fullmatch(r'[A-Za-z0-9+/]*={0,2}', text) and len(text) % 4 == 0:
        try:
            out.append(('base64', base64.b64decode(text, validate=True)))
        except ValueError:
            pass
    if re.fullmatch(r'[A-Z2-7]*={0,6}', text) and len(text) % 8 == 0:
        try:
            out.append(('base32', base64.b32decode(text)))
        except ValueError:
            pass
    if raw.startswith(b'BZh'):
        out.append(('bz2', bz2.decompress(raw)))
    if raw.startswith((b'\x78\x01', b'\x78\x9c', b'\x78\xda')):
        out.append(('zlib', zlib.decompress(raw)))
    if text and all(ord(c) < 128 for c in text):
        rot13 = codecs.decode(text, 'rot_13').encode()
        # ROT13 has no structural signature. Add it only when the result has
        # a strong language/challenge marker; otherwise inspect it manually.
        if any(token in rot13.lower() for token in (b'the', b'flag', b'ctf{', b'http')):
            out.append(('rot13', rot13))
    try:
        ebcdic = raw.decode('cp500').encode()
        # EBCDIC likewise accepts every byte sequence; require a material
        # quality improvement before treating it as a candidate.
        if quality(ebcdic) > quality(raw) + 0.20:
            out.append(('ebcdic-cp500', ebcdic))
    except UnicodeError:
        pass
    return [(name, blob, quality(blob)) for name, blob in out]

def auto_decode(initial, max_layers=50):
    data = initial
    seen = set()
    trace = []
    for _ in range(max_layers):
        marker = data if isinstance(data, bytes) else data.encode()
        if marker in seen:
            break
        seen.add(marker)

        options = candidates(data)
        if not options:
            break
        name, decoded, score = max(options, key=lambda item: item[2])
        # Low-scoring candidates are ambiguous; stop for manual inspection.
        if score < 0.80:
            break
        trace.append((name, score))
        data = decoded
    return data, trace
```

Inspect the trace and every intermediate artifact. A magic header, valid UTF text, expected flag prefix, URL or challenge marker is positive evidence; an exception-free call alone is not. For challenge-specific wrappers such as `REAL_DATA_FOLLOWS:`, strip the wrapper only after confirming its semantics. Brainfuck and other esolangs should be detected and routed to their interpreter rather than treated as another byte codec.

### URL Encoding
```python
import urllib.parse
urllib.parse.unquote('hello%20world')
```

### ROT13 / Caesar
```bash
echo "uryyb" | tr 'a-zA-Z' 'n-za-mN-ZA-M'
```

**ROT13 patterns:** `gur` = "the", `synt` = "flag"

### Caesar Brute Force
```python
text = "Khoor Zruog"
for shift in range(26):
    decoded = ''.join(
        chr((ord(c) - 65 - shift) % 26 + 65) if c.isupper()
        else chr((ord(c) - 97 - shift) % 26 + 97) if c.islower()
        else c for c in text)
    print(f"{shift:2d}: {decoded}")
```

---

### 补充要点

```bash
# Base64
echo "encoded" | base64 -d

# Base32 (A-Z2-7=)
echo "OBUWG32D..." | base32 -d

# Hex
echo "68656c6c6f" | xxd -r -p

# ROT13
echo "uryyb" | tr 'a-zA-Z' 'n-za-mN-ZA-M'
```

**Identify by charset:**
- Base64: `A-Za-z0-9+/=`
- Base32: `A-Z2-7=` (no lowercase)
- Hex: `0-9a-fA-F`

See encodings-qr-and-esolangs.md for Caesar brute force, URL encoding, and full details.


---
## Structured and Low-Frequency Encodings

These formats are reversible representations rather than cryptographic primitives. They live in the Crypto raw tree because the knowledge-base policy groups encoding challenges with cryptography; a visible carrier alone does not make a challenge steganography.

### Gray Code Cyclic Encoding (EHAX 2026)

**模式（#808080）：** A circular wheel has five concentric bit tracks and 32 angular positions. Consecutive values must form a cyclic 5-bit Gray-code sequence, so adjacent positions differ in exactly one bit.

```python
def gray_code(n_bits):
    return [i ^ (i >> 1) for i in range(1 << n_bits)]

def rotate(seq, offset):
    offset %= len(seq)
    return seq[offset:] + seq[:offset]

sequence = gray_code(5)
rotated = rotate(sequence, 4)
```

An `N`-bit Gray code has `2^N` unique values and remains valid after cyclic rotation. If the decoded output is coherent but shifted by `ROT-k`, rotate the sequence start by the same offset. Map the innermost circle to bit 0 and the outermost to bit `N-1`, then read all tracks at each angle to form one value.

### Binary Tree Path Encoding

For zero-based heap indexing, a left step (`0`) maps `j -> 2*j + 1` and a right step (`1`) maps `j -> 2*j + 2`. Reverse the parent relation to recover the path:

```python
def decode_path(index):
    path = []
    while index:
        if index & 1:          # odd index: left child
            path.append('0')
            index = (index - 1) // 2
        else:                  # even index: right child
            path.append('1')
            index = (index - 2) // 2
    return ''.join(reversed(path))
```

### RFC 4042 UTF-9 Decoding (SECCON 2015)

RFC 4042 is an April Fools' specification for Unicode on 9-bit-byte systems. Each unit has a continuation bit followed by eight data bits; concatenate data portions until a unit with continuation bit 0.

```python
def decode_utf9(bits):
    chars = []
    i = 0
    while i + 9 <= len(bits):
        codepoint_bits = ''
        while i + 9 <= len(bits):
            continuation = bits[i] == '1'
            codepoint_bits += bits[i+1:i+9]
            i += 9
            if not continuation:
                break
        chars.append(chr(int(codepoint_bits, 2)))
    return ''.join(chars)

# Preserve leading zeroes when the source is octal/hex; pad to a 9-bit boundary.
raw_bits = bin(int(octal_data, 8))[2:]
raw_bits = raw_bits.zfill((len(raw_bits) + 8) // 9 * 9)
print(decode_utf9(raw_bits))
```

Hints such as `4042`, “nine-bit bytes” or `UTF-9` are strong identifiers. Do not discard leading zero bits during integer conversion.

### Pixel Color Binary Encoding (Break In 2016)

A visibly narrow image, often seven or eight pixels wide, may encode one ASCII character per row. Determine which color/brightness means 0 and 1, then test 7-bit and 8-bit grouping.

```python
from PIL import Image

img = Image.open('challenge.png').convert('RGB')
text = []
for y in range(img.height):
    bits = ''.join('1' if img.getpixel((x, y))[0] > 128 else '0'
                   for x in range(img.width))
    if len(bits) == 7:
        bits = '0' + bits
    if len(bits) == 8:
        text.append(chr(int(bits, 2)))
print(''.join(text))
```

Test individual channels as well as luminance. This is direct color-to-bit representation; if pixels instead conceal a second image through LSBs, permutations or reconstruction, pivot to the Stego image volume.

### TOPKEK Binary Encoding (Hack The Vote 2016)

`KEK` represents bit 0, `TOP` represents bit 1, and exclamation marks encode repetition. Confirm the exact convention from a known sample; in this variant the run length equals the number of `!`, with zero exclamation marks treated as one.

```python
def decode_topkek(encoded):
    bits = []
    for token in encoded.split():
        base = token.rstrip('!')
        repeats = max(1, len(token) - len(base))
        if base == 'KEK':
            bits.append('0' * repeats)
        elif base == 'TOP':
            bits.append('1' * repeats)
    stream = ''.join(bits)
    return ''.join(chr(int(stream[i:i+8], 2))
                   for i in range(0, len(stream) - 7, 8))
```

### DTMF Digits to Multi-Tap Keypad Text (h4ckc0n 2017)

After extracting a DTMF digit stream, repeated presses on the same phone key can encode letters. Tone detection and custom frequency grids are covered in [audio-frequency-and-archive-stego.md](../stego/audio-frequency-and-archive-stego.md); this section handles the reversible digit-to-text layer.

```python
MULTITAP = {
    '2':'a', '22':'b', '222':'c',
    '3':'d', '33':'e', '333':'f',
    '4':'g', '44':'h', '444':'i',
    '5':'j', '55':'k', '555':'l',
    '6':'m', '66':'n', '666':'o',
    '7':'p', '77':'q', '777':'r', '7777':'s',
    '8':'t', '88':'u', '888':'v',
    '9':'w', '99':'x', '999':'y', '9999':'z',
}

def decode_multitap(groups):
    return ''.join(MULTITAP.get(group, '?') for group in groups)
```

Pauses delimit groups, especially between two letters that use the same key. Preserve timing gaps while extracting DTMF; flattening the sequence destroys those boundaries.

---
## QR Codes

### Basic Commands
```bash
zbarimg qrcode.png           # Decode
zbarimg -S*.enable qr.png    # All barcode types
qrencode -o out.png "data"   # Encode
```

### QR Structure

**Finder patterns (3 corners):** 7x7 modules at top-left, top-right, bottom-left

**Version formula:** `(version * 4) + 17` modules per side

### Repairing Damaged QR

```python
from PIL import Image
import numpy as np

img = Image.open('damaged_qr.png')
arr = np.array(img)

# Convert to binary
gray = np.mean(arr, axis=2)
binary = (gray < 128).astype(int)

# Find QR bounds
rows = np.any(binary, axis=1)
cols = np.any(binary, axis=0)
rmin, rmax = np.where(rows)[0][[0, -1]]
cmin, cmax = np.where(cols)[0][[0, -1]]

# Check finder patterns
qr = binary[rmin:rmax+1, cmin:cmax+1]
print("Top-left:", qr[0:7, 0:7].sum())  # Should be ~25
```

### Finder Pattern Template
```python
finder_pattern = [
    [1,1,1,1,1,1,1],
    [1,0,0,0,0,0,1],
    [1,0,1,1,1,0,1],
    [1,0,1,1,1,0,1],
    [1,0,1,1,1,0,1],
    [1,0,0,0,0,0,1],
    [1,1,1,1,1,1,1],
]
```

### QR Code Chunk Reassembly (LACTF 2026)

**模式（error-correction）：** QR code split into grid of chunks (e.g., 5x5 of 9x9 pixels), shuffled.

**Tools:** `segno` (Python QR library), `zbarimg` for decoding.

### QR Code Chunk Reassembly via Indexed Directories (UTCTF 2026)

**模式（QRecreate）：** QR code split into numbered chunks stored in separate directories. Directory names encode the chunk index as base64 (e.g., `MDAx` → `001` → index 1).

```python
import os, base64, math
from PIL import Image

# 1. Decode directory names to get indices
chunks = []
for dirname in os.listdir('chunks/'):
    index = int(base64.b64decode(dirname).decode())
    tile = Image.open(f'chunks/{dirname}/tile.png')
    chunks.append((index, tile))

# 2. Sort by index and arrange in grid
chunks.sort(key=lambda x: x[0])
n = len(chunks)
side = int(math.isqrt(n))
tile_w, tile_h = chunks[0][1].size

canvas = Image.new("RGB", (side * tile_w, side * tile_h), (255, 255, 255))
for i, (_, tile) in enumerate(chunks):
    r, c = divmod(i, side)
    canvas.paste(tile, (c * tile_w, r * tile_h))

canvas.save('reconstructed_qr.png')
# 3. Decode with zbarimg or pyzbar
```

**关键结论：** Unlike the LACTF variant (shuffled chunks requiring structural analysis), indexed chunks just need sorting. The challenge is recognizing that directory names are base64-encoded indices. Check `base64 -d` on folder names when they look like random strings.

---

### 补充要点

```bash
zbarimg qrcode.png       # Decode
qrencode -o out.png "data"
```

### Hexadecimal Sudoku + QR Distribution (BSidesSF 2026)

**模式（hexhaustion）：** Four QR codes contain the quadrants of a 16×16 hexadecimal Sudoku. Decode and assemble the quadrants, solve the grid with symbols `0-F`, then read the main diagonal as hex nibbles and pair them into ASCII bytes.

```python
from itertools import product

def solve_hex_sudoku(grid):
    digits = set(range(16))

    def possible(row, col):
        used = set(grid[row])
        used.update(grid[r][col] for r in range(16))
        box_row, box_col = (row // 4) * 4, (col // 4) * 4
        used.update(grid[r][c]
                    for r in range(box_row, box_row + 4)
                    for c in range(box_col, box_col + 4))
        used.discard(-1)
        return digits - used

    def solve():
        empty = next(((r, c) for r, c in product(range(16), repeat=2)
                      if grid[r][c] == -1), None)
        if empty is None:
            return True
        row, col = empty
        for value in possible(row, col):
            grid[row][col] = value
            if solve():
                return True
        grid[row][col] = -1
        return False

    if not solve():
        raise ValueError('unsatisfiable grid')
    return grid

solved = solve_hex_sudoku(grid)
diagonal = ''.join(format(solved[i][i], 'X') for i in range(16))
print(bytes.fromhex(diagonal).decode())
```

The QR codes are distribution containers, not the main obstacle. Recognize the pattern from four quadrants, hexadecimal symbols, blanks and 4×4 sub-boxes; verify the solved grid before extracting the diagonal.

### MaxiCode 2D Barcode Decoding (CSAW CTF 2016)

MaxiCode uses a central bullseye and a hexagonal module grid. Ordinary QR-only readers and `pyzbar` builds commonly do not support it; substituting an unrelated one-dimensional barcode reader does not decode MaxiCode.

```bash
# ZXing's Java CommandLineRunner includes a native MaxiCode reader.
java -cp javase.jar:core.jar \
  com.google.zxing.client.j2se.CommandLineRunner maxicode.gif
```

Identify the format before choosing a decoder: QR has three square finder patterns, while MaxiCode has concentric circles at the center. Keep the original-resolution image and avoid resampling the hexagonal grid before decoding.

See this file's QR structure, repair and chunk-reassembly sections for ordinary QR workflows; use the MaxiCode-specific route only after confirming its bullseye and hexagonal modules.


---
## Multi-Stage URL Encoding Chain (UTCTF 2026)

**模式（Breadcrumbs）：** Flag is hidden behind a chain of URLs, each encoded differently. Follow the breadcrumbs across external resources (GitHub Gists, Pastebin, etc.), decoding at each hop.

**Common encoding layers per hop:**
1. **Base64** → URL to next resource
2. **Hex** → URL to next resource (e.g., `68747470733a2f2f...` = `https://...`)
3. **ROT13** → final flag

**Decoding workflow:**
```python
import base64, codecs

# Hop 1: Base64
hop1 = "aHR0cHM6Ly9naXN0Lmdp..."
url2 = base64.b64decode(hop1).decode()

# Hop 2: Hex-encoded URL
hop2 = "68747470733a2f2f..."
url3 = bytes.fromhex(hop2).decode()

# Hop 3: ROT13-encoded flag
hop3 = "hgsynt{...}"
flag = codecs.decode(hop3, 'rot_13')
```

**关键结论：** Each resource contains a hint about the next encoding (e.g., "Three letters follow" hints at 3-character encoding like hex). Look for contextual clues in surrounding text (poetry, comments, filenames) that indicate the encoding type.

---

## Esoteric Languages

| Language | Pattern |
|----------|---------|
| Brainfuck | `++++++++++[>+++++++>` |
| Whitespace | Only spaces, tabs, newlines (or S/T/L substitution) |
| Ook! | `Ook. Ook? Ook!` |
| Malbolge | Extremely obfuscated |
| Piet | Image-based |

### Whitespace Language Parser (BYPASS CTF 2025)

**模式（Whispers of the Cursed Scroll）：** File contains only S (space), T (tab), L (linefeed) characters — or visible substitutes. Stack-based virtual machine (VM) with PUSH, OUTPUT, and EXIT instructions.

**Instruction set (IMP = Instruction Modification Parameter):**
| Instruction | Encoding | Action |
|-------------|----------|--------|
| PUSH | `S S` + sign + binary + `L` | Push number to stack (S=0, T=1, L=terminator) |
| OUTPUT CHAR | `T L S S` | Pop stack, print as ASCII character |
| EXIT | `L L L` | Halt program |

```python
def solve_whitespace(content):
    # Convert to S/T/L tokens (handle both raw whitespace and visible chars)
    if any(c in content for c in 'STL'):
        code = [c for c in content if c in 'STL']
    else:
        code = [{'\\s': 'S', '\\t': 'T', '\\n': 'L'}.get(c, '') for c in content]
        code = [c for c in code if c]

    stack, output, i = [], "", 0

    while i < len(code):
        if code[i:i+2] == ['S', 'S']:  # PUSH
            i += 2
            sign = 1 if code[i] == 'S' else -1
            i += 1
            val = 0
            while i < len(code) and code[i] != 'L':
                val = (val << 1) + (1 if code[i] == 'T' else 0)
                i += 1
            i += 1  # skip terminator L
            stack.append(sign * val)
        elif code[i:i+4] == ['T', 'L', 'S', 'S']:  # OUTPUT CHAR
            i += 4
            if stack:
                output += chr(stack.pop())
        elif code[i:i+3] == ['L', 'L', 'L']:  # EXIT
            break
        else:
            i += 1

    return output
```

---

### Custom Brainfuck Variants (Themed Esolangs)

**Standard Brainfuck operations to map:**
| Op | Meaning | Typical pattern |
|----|---------|-----------------|
| `+` | Increment cell | Most repeated word (defines values) |
| `-` | Decrement cell | Second most repeated word |
| `>` | Move pointer right | Short word, appears alone or with `.` |
| `<` | Move pointer left | Paired with `>` word |
| `[` | Begin loop | Appears at start of lines with `]` counterpart |
| `]` | End loop | Appears at end of same lines as `[` |
| `.` | Output char | Line terminator word |

```python
# Most frequent = likely + or -, line-ender = likely .

# Map words to BF ops, translate, run standard BF interpreter
mapping = {'arch': '+', 'linux': '-', 'i': '>', 'use': '<',
           'the': '[', 'way': ']', 'btw': '.'}
bf = ''.join(mapping.get(w, '') for w in words)
# Then execute bf string with a standard Brainfuck interpreter
```

**Real example (0xL4ugh CTF - "iUseArchBTW"):** `.archbtw` extension, "I use Arch Linux BTW" meme theme.

**Tips:** Try swapping `+`/`-` or `>`/`<` if output is not ASCII. Verify output starts with known flag format.

---

### Multi-Layer Esoteric Language Chains (Break In 2016)

Challenges may stack multiple esoteric languages requiring sequential interpretation:

1. **Piet:** Visual programming language using colored pixel blocks. Execute PNG images as code:
```bash
npiet challenge.png         # npiet interpreter
# Or: java -jar PietDev.jar challenge.png
```

2. **Malbolge:** Extremely difficult esoteric language. Decode output from previous layer:
```bash
# Piet output → base64 decode → Malbolge source
echo "piet_output" | base64 -d > program.mal
malbolge program.mal        # Or use online interpreter
```

Common esoteric chains: Piet → base64 → Malbolge, Brainfuck → Ook → Whitespace, JSFuck → standard JS.

**关键结论：** When a PNG file doesn't contain obvious visual stego, try interpreting it as Piet code. Use `file` + visual inspection to identify the first layer, then decode sequentially.

#### Brainfuck to Piet Thumbnail Pipeline (RITSEC 2018)

多层 esolang 的中间产物不一定是下一段源码，也可能是外部资源定位符。RITSEC 的变体先执行 Brainfuck 得到 YouTube URL，再取得缩略图、裁掉边框，最后把彩色块图作为 Piet 程序执行：

```bash
bf puzzle.bf                           # 输出 YouTube URL
yt-dlp --write-thumbnail --skip-download "$URL"
python crop_border.py thumbnail.jpg    # 保留 Piet 彩色块网格
npiet piet.png
```

每一步都要保存产物并重新识别格式；URL、图片和代码是三个不同证据层。Brainfuck 负责表示层解码，缩略图裁剪属于载体恢复，真正执行 Piet 前应确认色块尺寸和边界没有被 JPEG/缩放破坏。

**关键结论：** 不要假设 esolang 链只在文本之间转换；解释器输出 URL 时，应获取对应资源并重新分流，再继续下一层。

**参考：** RITSEC CTF 2018 — Brainfuck/Piet multi-layer challenge, writeup 12224


---

## IEEE-754 Float Encoding (Data Hiding)

**模式（Floating）：** Numbers are float32 values hiding raw bytes.

**关键结论：** A 32-bit float is just 4 bytes interpreted as a number. Reinterpret as raw bytes -> ASCII.

```python
import struct
floats = [1.234e5, -3.456e-7, ...]  # Whatever the challenge gives
flag = b''
for f in floats:
    flag += struct.pack('>f', f)
print(flag.decode())
```

**Variations:** Double `'>d'`, little-endian `'<f'`, mixed. See encodings-qr-and-esolangs.md for CyberChef recipe.


---

## Keyboard Shift Cipher

**模式（Frenzy）：** Characters shifted left/right on QWERTY keyboard layout.

**Decoding:** Use [dCode Keyboard Shift Cipher](https://www.dcode.fr/keyboard-shift-cipher) with automatic mode.


---

## Pigpen / Masonic Cipher

**模式（Working For Peanuts）：** Geometric symbols representing letters based on grid positions.

**Decoding:** Map symbols to Pigpen grid positions, or use online decoder.


---

## ASCII in Numeric Data Columns

**模式（Cooked Books）：** CSV/spreadsheet numeric values (48-126) are ASCII character codes.

```python
import csv
with open('data.csv') as f:
    reader = csv.DictReader(f)
    flag = ''.join(chr(int(row['Times Borrowed'])) for row in reader)
print(flag)
```

**CyberChef:** "From Decimal" recipe with line feed delimiter.


---

## Unicode Steganography

### Variation Selectors Supplement (U+E0100-U+E01EF)
**Patterns (Seen & emoji, Nullcon 2026):** Invisible Variation Selector Supplement characters encode ASCII via codepoint offset.

```python
# Extract hidden data from variation selectors after visible character
data = open('README.md', 'r').read().strip()
hidden = data[1:]  # Skip visible emoji character
flag = ''.join(chr((ord(c) - 0xE0100) + 16) for c in hidden)
```

### Unicode Tags Block (U+E0000-U+E007F) (UTCTF 2026)

**模式（Hidden in Plain Sight）：** Invisible Unicode Tag characters embedded in URLs, filenames, or text. Each tag codepoint maps directly to an ASCII character by subtracting `0xE0000`. URL-encoded as 4-byte UTF-8 sequences (`%F3%A0%81%...`).

```python
import urllib.parse

url = "https://example.com/page#Title%20%F3%A0%81%B5%F3%A0%81%B4...Visible%20Text"
decoded = urllib.parse.unquote(urllib.parse.urlparse(url).fragment)

flag = ''.join(
    chr(ord(ch) - 0xE0000)
    for ch in decoded
    if 0xE0000 <= ord(ch) <= 0xE007F
)
print(flag)
```

**关键结论：** Unicode Tags (U+E0001-U+E007F) mirror ASCII 1:1 — subtract `0xE0000` to recover the original character. They render as zero-width invisible glyphs in most fonts. Unlike Variation Selectors (U+E0100+), these have a simpler offset calculation and appear in URL fragments, challenge titles, or filenames where the text looks normal but has suspiciously long byte length.


---

## UTF-16 Endianness Reversal

**模式（endians）：** Text "turned to Japanese" -- mojibake from UTF-16 endianness mismatch.

```python
# If encoded as UTF-16-LE but decoded as UTF-16-BE:
fixed = mojibake.encode('utf-16-be').decode('utf-16-le')
```
