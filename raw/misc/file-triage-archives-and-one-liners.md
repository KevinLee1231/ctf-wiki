# CTF Misc - File Triage, Archives and One-Liners

## 阅读定位

杂项题中的文件定性、压缩包、多层编码、hash 识别与常用一次性命令。


---

## General Tips


---

## File Type Detection


---

## Archive Extraction

### Nested Archive Script
```bash
while f=$(ls *.tar* *.gz *.bz2 *.xz *.zip *.7z 2>/dev/null|head -1) && [ -n "$f" ]; do
    7z x -y "$f" && rm "$f"
done
```


---

## Hash Identification

MD5: `0x67452301` | SHA-256: `0x6a09e667` | MurmurHash64A: `0xC6A4A7935BD1E995`


---

## Technique 速查s

- **PyInstaller:** `pyinstxtractor.py packed.exe`. See vm-z3-sandbox-and-game-basics.md for opcode remapping.
- **Marshal:** `marshal.load(f)` then `dis.dis(code)`. See vm-z3-sandbox-and-game-basics.md.
- **Python env RCE:** `PYTHONWARNINGS=ignore::antigravity.Foo::0` + `BROWSER="cmd"`. See vm-z3-sandbox-and-game-basics.md.
- **WASM patching:** `wasm2wat` -> flip minimax -> `wat2wasm`. See vm-z3-sandbox-and-game-basics.md.
- **Float precision:** Large multipliers amplify FP errors into exploitable fractions. See vm-z3-sandbox-and-game-basics.md.
- **K8s RBAC bypass:** SA token -> impersonate -> hostPath mount -> read secrets. See vm-z3-sandbox-and-game-basics.md.
- **Cookie checkpoint:** Save session cookies before guesses, restore on failure to brute-force without reset. See game-state-websocket-and-wasm.md.
- **Flask cookie game state:** `flask-unsign -d -c '<cookie>'` decodes unsigned Flask sessions, leaking game answers. See game-state-websocket-and-wasm.md.
- **WebSocket teleport:** Modify `player.x`/`player.y` in console, call verification function. See game-state-websocket-and-wasm.md.
- **Time-only validation:** Start session, `time.sleep(required_seconds)`, submit win. See game-state-websocket-and-wasm.md.
- **Quine context detection:** Dual-purpose quine that prints itself (passes validation) and runs payload only in server process via globals gate. See pyjails.md.
- **Repunit decomposition:** Decompose target integer into sum of repunits (1, 11, 111, ...) using only 2 characters (`1` and `+`) for restricted eval. See pyjails.md.
- **De Bruijn sequence:** B(k, n) contains all k^n possible n-length strings as substrings; linearize by appending first n-1 chars. See game-state-websocket-and-wasm.md.
- **Brainfuck instrumentation:** Instrument BF interpreter to track tape cells, brute-force flag character-by-character via validation cell. See game-state-websocket-and-wasm.md.
- **WASM memory manipulation:** Patch WASM linear memory at runtime to set game state variables directly, bypassing game logic. See game-state-websocket-and-wasm.md.
- **Lua sandbox escape:** Bypass `load()`/`os.execute()` filters via `os["execute"]` table indexing or `loadstring` alias. See vm-z3-sandbox-and-game-basics.md.
- **C code jail via emoji + gadget embedding:** When only emoji and punctuation are allowed in C, use `(😃==😃)` as constant 1, build integers, embed gadgets in `add eax, imm32` constants, jump to offset+1 for shellcode primitives. See interactive-containers-jails-and-solvers.md.
- **Emulator ROM-switching:** `/load` replaces ROM but preserves CPU state (registers, RAM, PC). Switch ROMs at specific PCs to combine INIT from one ROM with display instructions from another → read protected memory. See interactive-containers-jails-and-solvers.md.
- **BuildKit daemon exploitation:** Exposed BuildKit gRPC allows nested `buildctl build` with `--mount=type=secret` to read build secrets. Two-stage Dockerfile: install buildctl → submit nested build mounting flag secret. See interactive-containers-jails-and-solvers.md.
- **Docker container escape:** Privileged breakout via host device mount, docker.sock socket escape, CAP_SYS_ADMIN cgroup release_agent, container info leakage via /proc and overlayfs. See interactive-containers-jails-and-solvers.md.
- **Taint analysis bypass via type coercion:** In custom ML-like languages with secrecy/taint systems, if-expression secrecy depends on return type not condition — coerce side-effecting functions to private type to leak private data through public mutable refs. See interactive-containers-jails-and-solvers.md.
- **Shredded document pixel-edge reassembly:** Encode each strip's left/right edge as binary bitmask (dark=1), use XOR + popcount Hamming distance to greedily place strips by minimum edge distance for sub-second reassembly. See interactive-containers-jails-and-solvers.md.
- **f-string config injection via stored eval:** Store payload as config value, create key named `eval(stored_key)` — f-string rendering evaluates the key name expression, triggering RCE. See pyjails.md.
- **Hexadecimal Sudoku + QR assembly:** 4 QR codes encode 16x16 hex Sudoku quadrants; solve grid, read diagonal as hex pairs → ASCII flag. See exotic-encodings-and-file-formats.md.
- **Z3 boolean gate network SAT solving:** Product key validation as 250 boolean gates (AND/OR/XOR/NOT) over 125 input bits. Model each gate as Z3 constraint, require all outputs True, solve in milliseconds. See vm-z3-sandbox-and-game-basics.md.


---

## Discord API Enumeration (0xFun 2026)

Flags hidden in Discord metadata (roles, animated emoji, embeds). Invoke `/ctf-osint` for Discord API enumeration technique and code (see social-media.md in ctf-osint).

---


---

## Useful One-Liners

```bash
grep -rn "flag{" .
strings file | grep -i flag
python3 -c "print(int('deadbeef', 16))"
```


---

## Cipher Identification Workflow

1. **ROT13** - Challenge mentions "ROT", text looks like garbled English
2. **Base64** - `A-Za-z0-9+/=`, title hints "64"
3. **Base32** - `A-Z2-7=` uppercase only
4. **Atbash** - Title hints (Abash/Atbash), preserves spaces, 1:1 substitution
5. **Pigpen** - Geometric symbols on grid
6. **Keyboard Shift** - Text looks like adjacent keys pressed
7. **Substitution** - Frequency analysis applicable

**Auto-identify:** [dCode Cipher Identifier](https://www.dcode.fr/cipher-identifier)
