# CTF Crypto - RSA Attacks

## 阅读定位

- 这是 RSA 的 **主卷**：先处理小指数、共模、Wiener、Fermat、Coppersmith、常见 oracle 和基础结构问题。
- 如果你遇到的是 CRT fault、ROCA、特殊模数关系、签名伪造或非常规 RSA 长尾，再读 rsa-specialized-structures-and-oracles.md。

## Table of Contents
- [Small Public Exponent (Cube Root)](#small-public-exponent-cube-root)
- [Common Modulus Attack](#common-modulus-attack)
- [Wiener's Attack (Small Private Exponent)](#wieners-attack-small-private-exponent)
- [Pollard's p-1 Factorization](#pollards-p-1-factorization)
- [Hastad's Broadcast Attack](#hastads-broadcast-attack)
- [RSA with Consecutive Primes (Fermat Factorization)](#rsa-with-consecutive-primes-fermat-factorization)
- [Multi-Prime RSA](#multi-prime-rsa)
- [RSA with Restricted-Digit Primes (LACTF 2026)](#rsa-with-restricted-digit-primes-lactf-2026)
- [Coppersmith for Structured RSA Primes (LACTF 2026)](#coppersmith-for-structured-rsa-primes-lactf-2026)
- [Manger's RSA Padding Oracle Attack (Nullcon 2026)](#mangers-rsa-padding-oracle-attack-nullcon-2026)
- [Manger's Attack on RSA-OAEP via Timing Oracle (HTB Early Bird)](#mangers-attack-on-rsa-oaep-via-timing-oracle-htb-early-bird)
- [Polynomial Hash with Trivial Root (Pragyan 2026)](#polynomial-hash-with-trivial-root-pragyan-2026)
- [Polynomial CRT in GF(2)\[x\] (Nullcon 2026)](#polynomial-crt-in-gf2x-nullcon-2026)
- [Affine Cipher over Non-Prime Modulus (Nullcon 2026)](#affine-cipher-over-non-prime-modulus-nullcon-2026)
- [Hastad Broadcast Attack with Linear Padding -- Coppersmith (PlaidCTF 2017)](#hastad-broadcast-attack-with-linear-padding----coppersmith-plaidctf-2017)
- [Franklin-Reiter Related Message Attack on RSA e=3 (N1CTF 2018)](#franklin-reiter-related-message-attack-on-rsa-e3-n1ctf-2018)
- [Coppersmith Attack on Linearly-Related RSA Primes (ASIS CTF 2018)](#coppersmith-attack-on-linearly-related-rsa-primes-asis-ctf-2018)
- rsa-specialized-structures-and-oracles.md: RSA p=q Validation Bypass (BearCatCTF 2026)
- rsa-specialized-structures-and-oracles.md: RSA Cube Root CRT when gcd(e, phi) > 1 (BearCatCTF 2026)
- rsa-specialized-structures-and-oracles.md: Factoring n from Multiple of phi(n) (BearCatCTF 2026)
- rsa-specialized-structures-and-oracles.md: RSA Signature Forgery via Multiplicative Homomorphism (MMA CTF 2015)
- rsa-specialized-structures-and-oracles.md: Weak RSA Key Generation via Base Representation (Sharif CTF 2016)
- rsa-specialized-structures-and-oracles.md: RSA with gcd(e, phi(n)) > 1 (CSAW 2015)
- rsa-specialized-structures-and-oracles.md: Batch GCD for Shared Prime Factoring (BSidesSF 2025)
- rsa-specialized-structures-and-oracles.md: RSA Partial Key Recovery from dp dq qinv (0CTF 2016)
- rsa-specialized-structures-and-oracles.md: RSA-CRT Fault Attack / Bit-Flip Recovery (CSAW CTF 2016)
- rsa-specialized-structures-and-oracles.md: RSA Homomorphic Decryption Oracle Bypass (ECTF 2016)
- rsa-specialized-structures-and-oracles.md: RSA with Small Prime Factors and CRT Decomposition (Hack The Vote 2016)

---

## Step 0: FactorDB 预查

**任何 RSA 题拿到 `n` 后第一步：** 查询 FactorDB 是否已有分解记录。

```
MCP tool: factordb_query(number="<n>")
```

- 状态 `FF` → 已完全分解，直接取因子得 p、q
- 状态 `CF` → 已知部分因子，可用于进一步分解
- 状态 `C` / `U` → 无记录，继续下方攻击流程

若自己分解出因子，通过 `factordb_report_factor(number="<n>", factor="<factor>")` 贡献回数据库。

---

## Small Public Exponent (Cube Root)

```python
import gmpy2

def small_e_attack(c, e):
    """Recover plaintext when m^e < n (no modular wrap)."""
    m, exact = gmpy2.iroot(c, e)
    if exact:
        return int(m)
    return None

# Usage
m = small_e_attack(c, e=3)
print(bytes.fromhex(hex(m)[2:]))
```

---

## Common Modulus Attack

```python
from math import gcd

def common_modulus_attack(c1, c2, e1, e2, n):
    """Recover plaintext from two encryptions with same n, coprime e1/e2."""
    # Extended GCD: find a, b such that a*e1 + b*e2 = 1
    def extended_gcd(a, b):
        if a == 0: return b, 0, 1
        g, x, y = extended_gcd(b % a, a)
        return g, y - (b // a) * x, x

    g, a, b = extended_gcd(e1, e2)
    assert g == 1, "e1 and e2 must be coprime"

    # m = c1^a * c2^b mod n
    # Handle negative exponent by using modular inverse
    if a < 0:
        c1 = pow(c1, -1, n)
        a = -a
    if b < 0:
        c2 = pow(c2, -1, n)
        b = -b
    m = (pow(c1, a, n) * pow(c2, b, n)) % n
    return m
```

---

## Wiener's Attack (Small Private Exponent)

```python
def wiener_attack(e, n):
    """Recover d when d < N^0.25 using continued fraction expansion of e/n."""
    def continued_fraction(num, den):
        cf = []
        while den:
            q, r = divmod(num, den)
            cf.append(q)
            num, den = den, r
        return cf

    def convergents(cf):
        convs = []
        h0, h1 = 0, 1
        k0, k1 = 1, 0
        for a in cf:
            h0, h1 = h1, a * h1 + h0
            k0, k1 = k1, a * k1 + k0
            convs.append((h1, k1))
        return convs

    cf = continued_fraction(e, n)
    for k, d in convergents(cf):
        if k == 0:
            continue
        # Check if d is valid: phi = (e*d - 1) / k must be integer
        if (e * d - 1) % k != 0:
            continue
        phi = (e * d - 1) // k
        # phi = (p-1)(q-1) = n - p - q + 1, so p+q = n - phi + 1
        s = n - phi + 1
        # p and q are roots of x^2 - s*x + n = 0
        discriminant = s * s - 4 * n
        if discriminant < 0:
            continue
        from math import isqrt
        t = isqrt(discriminant)
        if t * t == discriminant:
            return d
    return None

# Usage
d = wiener_attack(e, n)
m = pow(c, d, n)
```

**适用条件：** Very large `e` (close to `n`) often indicates small `d`. Also try `owiener` Python package: `pip install owiener`.

---

## Pollard's p-1 Factorization

```python
from math import gcd

def pollard_p1(n, B=100000):
    """Factor n when p-1 is B-smooth for some prime factor p."""
    a = 2
    for j in range(2, B + 1):
        a = pow(a, j, n)
        d = gcd(a - 1, n)
        if 1 < d < n:
            return d, n // d
    return None

# Usage
result = pollard_p1(n)
if result:
    p, q = result
```

**关键结论：** By Fermat's little theorem, if `p-1` divides `B!`, then `a^(B!) ≡ 1 (mod p)`, so `gcd(a^(B!) - 1, n)` gives `p`. Increase `B` for larger smooth bounds. CTF primes generated with `getStrongPrime()` or similar are resistant.

---

## Hastad's Broadcast Attack

```python
from functools import reduce

def hastad_broadcast(ciphertexts, moduli, e):
    """Recover m from e encryptions with the same exponent e."""
    assert len(ciphertexts) >= e and len(moduli) >= e

    # Chinese Remainder Theorem
    def crt(remainders, moduli):
        N = reduce(lambda a, b: a * b, moduli)
        result = 0
        for r, m in zip(remainders, moduli):
            Ni = N // m
            Mi = pow(Ni, -1, m)
            result += r * Ni * Mi
        return result % N

    # CRT gives m^e (mod N1*N2*...*Ne)
    # Since m < each Ni, m^e < N1*N2*...*Ne, so no modular reduction occurred
    me = crt(ciphertexts[:e], moduli[:e])

    import gmpy2
    m, exact = gmpy2.iroot(me, e)
    if exact:
        return int(m)
    return None

# Usage (e=3, three encryptions)
m = hastad_broadcast([c1, c2, c3], [n1, n2, n3], e=3)
print(bytes.fromhex(hex(m)[2:]))
```

**关键结论：** CRT reconstructs `m^e` exactly (no modular reduction) because `m < min(n_i)` and therefore `m^e < n_1 * n_2 * ... * n_e`. Taking the integer eth root recovers `m`.

---

## Hastad Broadcast Attack with Linear Padding -- Coppersmith (PlaidCTF 2017)

```python
# Standard Hastad requires identical plaintext
# With linear padding: each ciphertext encrypts a_i*m + b_i
# Use CRT + Coppersmith's small_roots on the resulting polynomial

from sage.all import *
# Combine via CRT
N = prod(n_values)
T = [crt_coefficient(i, n_values) for i in range(e)]

P = PolynomialRing(Zmod(N), 'x')
x = P.gen()
poly = sum(T[i] * ((a[i]*x + b[i])**e - c[i]) for i in range(e))
poly = poly.monic()

# Coppersmith's method finds small root
roots = poly.small_roots(epsilon=1/30)
flag = int(roots[0])
```

**关键结论：** When the same message is encrypted with `e` different moduli but each applies a known affine transform `a_i * m + b_i`, CRT combines the congruences into a single polynomial of degree `e` over `Z/NZ`. Coppersmith's method recovers `m` as a small root, generalizing Hastad's attack beyond identical plaintexts.

**参考：** PlaidCTF 2017

---

### Franklin-Reiter Related Message Attack on RSA e=3 (N1CTF 2018)

```python
# SageMath
def franklin_reiter(n, pad1, pad2, c1, c2):
    R.<X> = PolynomialRing(Zmod(n))
    f1 = (X + pad1)^3 - c1
    f2 = (X + pad2)^3 - c2
    return -gcd(f1, f2).coefficients()[0]
```

**关键结论：** With RSA e=3, if the same message `m` is encrypted with two known affine transformations (`m+pad1`, `m+pad2`), polynomial GCD over `Zmod(n)` recovers `m` directly. Works whenever the padding difference is known, even without knowing the full padding.

---

### Coppersmith Attack on Linearly-Related RSA Primes (ASIS CTF 2018)

```python
# SageMath
qbar = isqrt(4 * n)
R.<x> = PolynomialRing(Zmod(n))
f = x + qbar
roots = f.small_roots(X=2^200, beta=0.5)  # find small error term
q = qbar + int(roots[0])
p = n // q
```

**关键结论：** When `q ~ k*p` for known `k`, then `q ~ sqrt(k*n)`. The difference between `q` and this approximation is small enough for Coppersmith's method. This generalizes Fermat factorization to non-consecutive primes with known ratio.

---

## RSA with Consecutive Primes (Fermat Factorization)

**模式（Loopy Primes）：** q = next_prime(p), making p ~ q ~ sqrt(N). Also known as Fermat factorization — works whenever `|p - q|` is small.

**Factorization:** Find first prime below sqrt(N):
```python
from sympy import nextprime, prevprime, isqrt

root = isqrt(n)
p = prevprime(root + 1)
while n % p != 0:
    p = prevprime(p)
q = n // p
```

**Multi-layer variant:** 1024 nested RSA encryptions, each with consecutive primes of increasing bit size. Decrypt in reverse order.

---

## Multi-Prime RSA

When N is product of many small primes (not just p*q):
```python
# Factor N (easier when many primes)
from sympy import factorint
factors = factorint(n)  # Returns {p1: e1, p2: e2, ...}

# Compute phi using all factors
phi = 1
for p, e in factors.items():
    phi *= (p - 1) * (p ** (e - 1))

d = pow(e, -1, phi)
plaintext = pow(ciphertext, d, n)
```

---

## RSA with Restricted-Digit Primes (LACTF 2026)

**模式（six-seven）：** RSA primes p, q composed only of digits {6, 7}, ending in 7.

**Digit-by-digit factoring from LSB:**
```python
# At each step k, we know p mod 10^k -> compute q mod 10^k = n * p^{-1} mod 10^k
# Prune: only keep candidates where digit k of both p and q is in {6, 7}
candidates = [(6,), (7,)]  # p ends in 6 or 7
for k in range(1, num_digits):
    new_candidates = []
    for p_digits in candidates:
        for d in [6, 7]:
            p_val = sum(p_digits[i] * 10**i for i in range(len(p_digits))) + d * 10**k
            q_val = (n * pow(p_val, -1, 10**(k+1))) % 10**(k+1)
            q_digit_k = (q_val // 10**k) % 10
            if q_digit_k in {6, 7}:
                new_candidates.append(p_digits + (d,))
    candidates = new_candidates
```

**General lesson:** When prime digits are restricted to a small set, digit-by-digit recovery from LSB with modular arithmetic prunes exponentially. Works for any restricted character set.

---

## Coppersmith for Structured RSA Primes (LACTF 2026)

**模式（six-seven-again）：** p = base + 10^k * x where base is fully known and x is small (x < N^0.25).

**Attack via SageMath:**
```python
# Construct f(x) such that f(x_secret) = 0 (mod p) and thus (mod N)
# p = base + 10^k * x -> x + base * (10^k)^{-1} = 0 (mod p)
R.<x> = PolynomialRing(Zmod(N))
f = x + (base * inverse_mod(10**k, N)) % N
roots = f.small_roots(X=2**70, beta=0.5)  # x < N^0.25
```

**适用条件：** Whenever part of a prime is known and the unknown part is small enough for Coppersmith bounds (< N^{1/e} for degree-e polynomial, approximately N^0.25 for linear).

---

## Manger's RSA Padding Oracle Attack (Nullcon 2026)

**模式（TLS, Nullcon 2026）：** RSA-encrypted key with threshold oracle. Phase 1: double f until `k*f >= threshold`. Phase 2: binary search. ~128 total queries for 64-bit key.

See number-theory-and-algebra-attacks.md for full implementation.

---

## Manger's Attack on RSA-OAEP via Timing Oracle (HTB Early Bird)

**Vulnerable code pattern:**
```python
if Y != 0 or not self.H_verify(self.L, DB[:self.hLen]) or self.os2ip(PS) != 0:
    return {"ok": False, "error": "decryption error"}
```

**Oracle mapping:** Fast response → Y != 0 (decrypted message >= B). Slow response → Y == 0 (decrypted message < B = 2^(8*(k-1))).

**Calibration for network reliability:**
```python
def calibrate(n, e, k):
    B = pow(2, 8 * (k - 1))
    slow_times, fast_times = [], []
    for i in range(5):
        # Known-slow: encrypt values < B
        enc = pow(B - 1 - i*100, e, n).to_bytes(k, 'big')
        slow_times.append(measure(enc))
        # Known-fast: encrypt values > B
        enc = pow(B + 1 + i*100, e, n).to_bytes(k, 'big')
        fast_times.append(measure(enc))
    FAST_UPPER = max(fast_times) * 1.5
    SLOW_LOWER = min(slow_times) * 0.9
```

**Oracle with retry for ambiguous results:**
```python
def padding_oracle(c_int):
    while True:
        total = measure_response_time(c_int)
        if SLOW_LOWER < total < SLOW_UPPER:
            return True   # Y == 0 (below B)
        elif total < FAST_UPPER:
            return False  # Y != 0 (above B)
        # Ambiguous: retry
```

**Full 3-step Manger's attack (~1024 iterations for 1024-bit RSA):**
```python
# Step 1: Find f1 where f1 * m >= B
f1 = 2
while oracle((pow(f1, e, n) * c) % n):
    f1 *= 2

# Step 2: Find f2 where n <= f2 * m < n + B
f2 = (n + B) // B * f1 // 2
while not oracle((pow(f2, e, n) * c) % n):
    f2 += f1 // 2

# Step 3: Binary search narrowing m to exact value
mmin, mmax = ceil_div(n, f2), floor_div(n + B, f2)
while mmin < mmax:
    f = floor_div(2 * B, mmax - mmin)
    i = floor_div(f * mmin, n)
    f3 = ceil_div(i * n, mmin)
    if oracle((pow(f3, e, n) * c) % n):
        mmax = floor_div(i * n + B, f3)
    else:
        mmin = ceil_div(i * n + B, f3)
m = mmin
```

**Post-recovery OAEP decode:**
```python
from Crypto.Signature.pss import MGF1
maskedSeed = EM[1:hLen+1]
maskedDB = EM[hLen+1:]
seed = bytes(a ^ b for a, b in zip(maskedSeed, MGF1(maskedDB, hLen, HF)))
DB = bytes(a ^ b for a, b in zip(maskedDB, MGF1(seed, k - hLen - 1, HF)))
# DB[:hLen] should match lHash; rest is 0x00...0x01 || message
```

**关键结论：** Python's `or` short-circuits left-to-right. When expensive operations (PBKDF2, bcrypt, argon2) appear in chained conditions, the first condition becomes a timing oracle. RFC 8017 explicitly warns implementations must not let attackers distinguish error conditions — timing differences violate this.

---

## Polynomial Hash with Trivial Root (Pragyan 2026)

**模式（!!Cand1esaNdCrypt0!!）：** RSA signature scheme using polynomial hash `g(x,a,b) = x(x^2 + ax + b) mod P`.

**Vulnerability:** `g(0) = 0` for all parameters `a,b`. RSA signature of 0 is always 0 (`0^d mod n = 0`).

**Exploitation:** Craft message suffix so `bytes_to_long(prefix || suffix) = 0 (mod P)`:
```python
P = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF61  # 128-bit prime
# Compute required suffix value mod P
req = (-prefix_val * pow(256, suffix_len, P)) % P
# Brute-force partial bytes until all printable ASCII
while True:
    high = os.urandom(32).translate(printable_table)
    low_val = (req - int.from_bytes(high, 'big') * shift) % P
    low = low_val.to_bytes(16, 'big')
    if all(32 <= b <= 126 for b in low):
        suffix = high + low
        break
# Signature is simply 0
```

**General lesson:** Always check if hash function has trivial inputs (0, 1, -1). Factoring the polynomial often reveals these.

---

## Polynomial CRT in GF(2)[x] (Nullcon 2026)

**模式（Going in Circles, Nullcon 2026）：** `r = flag mod f` where f is random GF(2) polynomial. Collect ~20 pairs, filter coprime, CRT combine.

See number-theory-and-algebra-attacks.md for GF(2)[x] polynomial arithmetic and CRT implementation.

---

## Affine Cipher over Non-Prime Modulus (Nullcon 2026)

**模式（Matrixfun, Nullcon 2026）：** `c = A @ p + b (mod m)` with composite m. Chosen-plaintext difference attack. For composite modulus, solve via CRT in each prime factor field separately.

See number-theory-and-algebra-attacks.md for CRT approach and Gauss-Jordan implementation.

See also: rsa-specialized-structures-and-oracles.md for specialized RSA techniques (p=q bypass, cube root CRT, phi(n) multiple factoring, signature forgery, weak keygen, batch GCD, partial key recovery, CRT fault attack, homomorphic bypass).


---

## RSA Attacks

- **Small e with small message:** Take eth root
- **Common modulus:** Extended GCD attack
- **Wiener's attack:** Small d
- **Fermat factorization:** p and q close together
- **Pollard's p-1:** Smooth p-1
- **Hastad's broadcast:** Same message, multiple e=3 encryptions
- **Consecutive primes:** q = next_prime(p); find first prime below sqrt(N)
- **Multi-prime:** Factor N with sympy; compute phi from all factors
- **Restricted-digit primes:** Digit-by-digit factoring from LSB with modular pruning
- **Coppersmith structured primes:** Partially known prime; `f.small_roots()` in SageMath
- **Manger oracle (simplified):** Phase 1 doubling + phase 2 binary search; ~128 queries for 64-bit key
- **Manger on RSA-OAEP (timing):** Python `or` short-circuit skips expensive PBKDF2 when Y != 0, creating fast/slow timing oracle. Full 3-step attack (~1024 iterations for 1024-bit RSA). Calibrate timing bounds with known-fast/known-slow samples.
- **Polynomial hash (trivial root):** `g(0) = 0` for polynomial hash; craft suffix for `msg = 0 (mod P)`, signature = 0
- **Polynomial CRT in GF(2)[x]:** Collect ~20 remainders `r = flag mod f`, filter coprime, CRT combine
- **Affine over composite modulus:** CRT in each prime factor field; Gauss-Jordan per prime
- **RSA p=q validation bypass:** Set `p=q` so server computes wrong `phi=(p-1)^2` instead of `p*(p-1)`; test decryption fails, leaking ciphertext
- **RSA cube root CRT (gcd(e,phi)>1):** When all primes ≡ 1 mod e, compute eth roots per-prime via `nthroot_mod`, enumerate CRT combinations (3^k feasible for small k)
- **Factoring from phi(n) multiple:** Any multiple of `phi(n)` (e.g., `e*d-1`) enables factoring via Miller-Rabin square root technique; succeeds with prob ≥ 1/2 per attempt
- **Weak keygen via base representation:** Primes `p = kp*B + tp` with small kp create mixed-radix structure in n; brute-force kp*kq (2^24) to factor
- **RSA with gcd(e,phi)>1 (exponent reduction):** Reduce `e' = e/g`, compute `d' = e'^(-1) mod phi`, partial decrypt to `m^g`, then take g-th root over integers
- **RSA partial key recovery (dp/dq/qinv):** CRT exponents from partial PEM leak allow O(e) prime recovery: iterate k, check if `(dp*e-1)/k+1` is prime. See rsa-specialized-structures-and-oracles.md.
- **RSA-CRT fault attack:** Single faulty CRT signature leaks factor via `gcd(s^e - m, n)` (Bellcore attack). See rsa-specialized-structures-and-oracles.md.
- **RSA homomorphic decryption bypass:** Multiplicative homomorphism lets you decrypt `c` by querying oracle with `c * r^e mod n`, then dividing result by `r`. See rsa-specialized-structures-and-oracles.md.
- **RSA small prime CRT decomposition:** When `n` has many small prime factors, factor with trial division, solve `m mod p_i` per prime, CRT combine. See rsa-specialized-structures-and-oracles.md.
- **Hastad broadcast with linear padding (Coppersmith):** When each of `e` recipients applies a known affine transform `a_i*m+b_i` before encryption, CRT + Coppersmith small_roots recovers `m`. See rsa-attacks.md.
- **RSA Montgomery reduction timing attack:** Leaked extra-subtraction counts in Montgomery multiplication reveal private key bits MSB-to-LSB via statistical correlation. See rsa-specialized-structures-and-oracles.md.
- **Bleichenbacher low-exponent signature forgery:** With e=3, forge PKCS#1 v1.5 signatures by computing cube root of a value with correct padding prefix; trailing garbage absorbs the remainder. See rsa-specialized-structures-and-oracles.md.
- **Franklin-Reiter related message attack (e=3):** Two ciphertexts of `m+pad1` and `m+pad2` with known padding difference; polynomial GCD in `Zmod(n)` recovers `m` directly. See rsa-attacks.md.
- **RSA signature bypass (e=1, crafted modulus):** Verifier accepts user-supplied `(n, e)`; set `e=1` and `n = sig - PKCS1_pad(msg)` so `pow(sig, 1, n)` equals expected padded hash. See rsa-specialized-structures-and-oracles.md.
- **Coppersmith on linearly-related primes:** When `q ~ k*p` for known `k`, approximate `q ~ sqrt(k*n)` and use Coppersmith `small_roots` on the error term. Generalizes Fermat factorization to non-consecutive primes. See rsa-attacks.md.

See rsa-attacks.md and number-theory-and-algebra-attacks.md for full code examples.
