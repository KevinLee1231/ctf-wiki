# CTF Crypto - PRNG Z3, LCG and Timing Attacks

Advanced CTF-specific PRNG attacks from 2017 onward. For foundational PRNG recovery (MT19937, LCG parameter recovery, ChaCha20, V8 XorShift128+, password cracking), see mt-lcg-and-seed-recovery.md.

## 阅读定位

- 本卷是 PRNG 的 **高级补卷**，偏近年 CTF 专题案例，不是首轮入口。
- 只有当 mt-lcg-and-seed-recovery.md 的基础模型已经不足以解释当前题目时，再读这里。


## Table of Contents
- [Mersenne Twister Seed Recovery from Subset Sum (Tokyo Westerns 2017)](#mersenne-twister-seed-recovery-from-subset-sum-tokyo-westerns-2017)
- [MT19937 State Recovery via Constraint Propagation (HITCON 2017)](#mt19937-state-recovery-via-constraint-propagation-hitcon-2017)
- [Rule 86 Cellular Automaton PRNG Reversal via Z3 (Insomni'hack 2018)](#rule-86-cellular-automaton-prng-reversal-via-z3-insomnihack-2018)
- [Java LCG Seed Meet-in-the-Middle via Partial Modulo (P.W.N. CTF 2018)](#java-lcg-seed-meet-in-the-middle-via-partial-modulo-pwn-ctf-2018)
- [LCG Backward Stepping via Multiplicative Inverse (P.W.N. CTF 2018)](#lcg-backward-stepping-via-multiplicative-inverse-pwn-ctf-2018)
- [LFSR Bit-Fold Recovery from ASCII Parity (X-MAS CTF 2018)](#lfsr-bit-fold-recovery-from-ascii-parity-x-mas-ctf-2018)
- [Z3 Solve-Time Timing Oracle on PRNG (X-MAS CTF 2018)](#z3-solve-time-timing-oracle-on-prng-x-mas-ctf-2018)
- [randcrack-Fed DSA k Prediction (CSAW CTF 2018)](#randcrack-fed-dsa-k-prediction-csaw-ctf-2018)
- [Time-Seeded PRNG Offset via Format-String Global Write (FireShell 2019)](#time-seeded-prng-offset-via-format-string-global-write-fireshell-2019)
- [NTP-Poisoned PRNG State Leak via UUID XOR (RuCTFe 2018)](#ntp-poisoned-prng-state-leak-via-uuid-xor-ructfe-2018)

---

## Mersenne Twister Seed Recovery from Subset Sum (Tokyo Westerns 2017)

**MT twist function relationship:**
```text
mt[i] = mt[i-624] XOR twist(mt[i-624], mt[i-623])
```
At the wrap-around: `mt[624]` depends on `mt[0]` (new cycle) and `mt[397]` (old cycle). Recovering `mt[0]` and `mt[227]` (which is related to `mt[624-227] = mt[397]`) via subset-sum solutions reveals enough to invert the twist recurrence.

```python
import random

def crack_seed_from_two_outputs(mt0_val, mt227_val):
    """Try all 2^32 seeds until MT outputs match recovered values."""
    for seed in range(2**32):
        r = random.Random()
        r.seed(seed)
        # Generate enough to reach indices 0 and 227
        outputs = [r.getrandbits(32) for _ in range(228)]
        if outputs[0] == mt0_val and outputs[227] == mt227_val:
            return seed
    return None

# After recovering seed, all future (and past) outputs are predictable
r = random.Random()
r.seed(recovered_seed)
```

**关键结论：** MT19937 seeds recoverable from as few as two state values (indices 0 and 227) via the twist function's wrap-around relationship. Any challenge that exposes MT state values through solvable mathematical puzzles is vulnerable to full seed recovery.

**参考：** Tokyo Westerns CTF 2017

---

## MT19937 State Recovery via Constraint Propagation (HITCON 2017)

**MT recurrence dependencies:**
```text
state[i] = state[i-624] XOR twist(state[i-624], state[i-623])
```
This means `state[x]` depends on `state[x-624]`, `state[x-623]`, and `state[x-227]` (via the generate step). Partial knowledge at any index propagates in both directions.

**Constraint propagation approach:**
```python
# Model: each state word starts as a set of 2^32 candidates
# Partial observation: narrow candidates for observed indices
# Propagate: for each constrained cell, narrow related cells

def propagate_forward(state_candidates, idx):
    """MT: state[idx+624] = f(state[idx], state[idx+1])"""
    for s0 in state_candidates[idx]:
        for s1 in state_candidates[idx + 1]:
            new_val = mt_twist(s0, s1)
            state_candidates[idx + 624].add(new_val)

def propagate_backward(state_candidates, idx):
    """Invert MT twist to constrain earlier states from later ones."""
    for val in state_candidates[idx]:
        # Recover state[idx-624] given state[idx] and state[idx-623]
        for s1 in state_candidates[idx - 623]:
            s0 = mt_untwist(val, s1)
            state_candidates[idx - 624].add(s0)

# After ~20 partial observations across different positions:
# Most cells converge to single candidates → full state determined
```

**关键结论：** MT19937's recurrence dependencies allow bidirectional constraint propagation — partial knowledge at multiple positions narrows candidates until the full 624-word state is determined. The number of partial observations needed scales inversely with bits leaked per observation: ~20 observations of 24+ bits each typically suffice.

**参考：** HITCON CTF 2017

---

## Rule 86 Cellular Automaton PRNG Reversal via Z3 (Insomni'hack 2018)

```python
from z3 import *

def RULE86(x, y, z):
    return Or(And(Not(x), Not(y), z), And(Not(x), y, Not(z)),
              And(x, Not(y), Not(z)), And(x, y, Not(z)))

s = Solver()
state = [Bool(f'b{i}') for i in range(256)]
# Forward-compute 128 rounds symbolically
for round in range(128):
    new_state = [RULE86(state[(i-1)%256], state[i], state[(i+1)%256]) for i in range(256)]
    state = new_state
# Constrain final state to known output
for i, bit in enumerate(known_output):
    s.add(state[i] == (bit == 1))
s.check()
model = s.model()
```

**关键结论：** Elementary cellular automata are NOT injective -- multiple preimages may exist. But Z3 handles the search efficiently by treating each cell as a boolean variable and each rule application as a CNF clause. For Rule 86 specifically, the DNF has 4 terms (bits 1,2,4,6 of rule number 86 = 01010110). Use `s.push()`/`s.pop()` to iteratively backtrack through rounds. This approach generalizes to any elementary CA rule used as a PRNG: encode the rule's truth table as a boolean formula, compose symbolically for N rounds, and constrain to the known output.

**参考：** Insomni'hack CTF 2018

---

## Java LCG Seed Meet-in-the-Middle via Partial Modulo (P.W.N. CTF 2018)

```python
# Phase 1: enumerate 2^18 low-18-bit candidates whose nextInt(62) parities match known chars
for low in range(1 << 18):
    if simulate(low)[: K] == known_prefix: candidates.append(low)

# Phase 2: extend each candidate to 48 bits, matching next outputs
for low in candidates:
    for high in range(1 << 30):
        seed = (high << 18) | low
        if simulate(seed) == full_known: return seed
```

**关键结论：** Java's LCG discards the lowest 16 bits, so the low 18 bits only affect the LSB of each `nextInt()`. Split the seed at that boundary to turn an infeasible `2^48` search into two `2^18 + 2^30` passes.

**参考：** P.W.N. CTF 2018 — PW API, writeup 12065

---

## LCG Backward Stepping via Multiplicative Inverse (P.W.N. CTF 2018)

```python
M = 1 << 48
a_inv = pow(25214903917, -1, M)  # Java multiplier inverse
prev_state = (a_inv * (state - 11)) % M
```

**关键结论：** Every power-of-two-modulus LCG is invertible; once you have any state, you can walk the chain both directions without re-simulating from seed.

**参考：** P.W.N. CTF 2018 — PW API, writeup 12065

---

## LFSR Bit-Fold Recovery from ASCII Parity (X-MAS CTF 2018)

```python
# Collect parity constraints: each observed ASCII byte gives top_bit == 0
# Each bit of each byte is a linear combination of state bits
# Stack rows, solve in GF(2)
import numpy as np
A = np.array(constraint_rows, dtype=np.uint8)
b = np.array(known_bits,      dtype=np.uint8)
state = gf2_solve(A, b)
```

**关键结论：** Output byte folding does not hide the state; each output bit is still linear in the state. Use the ASCII constraint to turn output bytes into free parity equations.

**参考：** X-MAS CTF 2018 — Probably Really Nice Goodies from Santa, writeup 12686

---

## Z3 Solve-Time Timing Oracle on PRNG (X-MAS CTF 2018)

```python
from z3 import Solver, sat
for c in string.printable:
    s = Solver()
    s.set('timeout', 500)
    s.add(prng_constraints(flag + c, ciphertext))
    t = time.time()
    if s.check() == sat and (time.time() - t) > 0.4:
        flag += c
        break
```

**关键结论：** Many oracles run the solver internally; wrong guesses return fast because the SAT instance is already obviously satisfiable. Set a tight timeout and promote slow-solve candidates.

**参考：** X-MAS CTF 2018 — Probably Really Nice Goodies from Santa, writeup 12686

---

## randcrack-Fed DSA k Prediction (CSAW CTF 2018)

```python
from randcrack import RandCrack
rc = RandCrack()
for _ in range(624 // 2):
    v = getrand64()
    rc.submit(v & 0xffffffff); rc.submit(v >> 32)
k = rc.predict_randrange(2, q)
x = ((s*k - h) * pow(r, -1, q)) % q
```

**关键结论：** Python's `random` is shared across all calls. Any endpoint that leaks `random` output poisons every other RNG consumer including DSA signing.

**参考：** CSAW CTF 2018 — Disastrous Security Apparatus, writeup 12495

---

## Time-Seeded PRNG Offset via Format-String Global Write (FireShell 2019)

```python
# 1. Format-string: %Xc%Y$n  writes desired bet value at &bet (0x602020)
# 2. Locally: predict rand() with C stdlib
from ctypes import CDLL
libc = CDLL('libc.so.6')
libc.srand((int(time.time())//10) + bet_value)
predicted = [libc.rand() for _ in range(n)]
```

**关键结论：** Shifting a known seed by a writable constant turns a time-of-day RNG into a deterministic RNG because the attacker now controls the whole seed.

**参考：** FireShell CTF 2019 — casino, writeup 12916

---

## NTP-Poisoned PRNG State Leak via UUID XOR (RuCTFe 2018)

```python
# 1. Point the server at attacker-controlled NTP that returns 0
# 2. Register; received UUID == internal state
# 3. For desired uuid, compute needed timestamp = uuid ^ state
# 4. Send the crafted timestamp and read the message
```

**关键结论：** Any randomness mixed via XOR with a user-controlled value is the same as giving the attacker the state directly. Check all time sources for user control.

**参考：** RuCTFe 2018 — vch, writeup 12146


---

## C srand/rand Prediction via ctypes (L3akCTF 2024, MireaCTF)


---

## Chaotic PRNG (Logistic Map)

- **Logistic map:** `x = r * x * (1 - x)`, `r ≈ 3.99-4.0`; seed recovery by brute-forcing high-precision decimals
- **Keystream:** `struct.pack("<f", x)` per iteration; XOR with ciphertext

See mt-lcg-and-seed-recovery.md for full code.
