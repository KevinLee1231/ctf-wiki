# CTF Crypto - Exotic Secret Sharing, Rabin and Polynomials

Covers the continuation-only cases that are not already present in homomorphic-and-exotic-algebra.md: Cayley-Purser, BIP39 brute force, Asmuth-Bloom, polynomial-prime Rabin, LCG period detection, Vandermonde recovery, and four-root Rabin decryption.

## 阅读定位

- 本卷是 homomorphic-and-exotic-algebra.md 的 **后续补卷**，偏长尾、偏案例库。
- 当你已经确认问题属于 exotic crypto，且主卷没有覆盖当前结构时再读这里。
- 如果题目只是普通同态、ElGamal、Paillier 或群结构入门，先从主卷开始。


## Table of Contents
- [Cayley-Purser Decryption Without Private Key (TJCTF 2018)](#cayley-purser-decryption-without-private-key-tjctf-2018)
- [BIP39 Partial-Mnemonic Brute Force via Checksum (SECCON 2018)](#bip39-partial-mnemonic-brute-force-via-checksum-seccon-2018)
- [Asmuth-Bloom Threshold Secret Sharing via CRT (X-MAS 2018)](#asmuth-bloom-threshold-secret-sharing-via-crt-x-mas-2018)
- [Rabin Cryptosystem with Polynomial Primes (X-MAS 2018)](#rabin-cryptosystem-with-polynomial-primes-x-mas-2018)
- [LCG Period Detection for Unlimited Output Prediction (X-MAS 2018)](#lcg-period-detection-for-unlimited-output-prediction-x-mas-2018)
- [Polynomial Coefficient Recovery via Vandermonde Linear System (X-MAS 2018)](#polynomial-coefficient-recovery-via-vandermonde-linear-system-x-mas-2018)
- [Rabin Decryption via Four-Roots CRT Combination (Pragyan CTF 2019)](#rabin-decryption-via-four-roots-crt-combination-pragyan-ctf-2019)

---

## Cayley-Purser Decryption Without Private Key (TJCTF 2018)

**Exploit:** Any matrix `H` commuting with `gamma` decrypts correctly, and the Cayley-Hamilton theorem lets you build one entirely from public values — no need to recover `r`.

```python
from sage.all import matrix, identity_matrix
import operator

# Given public alpha, beta, gamma, epsilon, mu (ciphertext)
invalpha = alpha.inverse()
# Recover scaling entry h via elementwise division
h_elems = (invalpha * gamma - gamma * beta)
h_denom = (beta - invalpha)
h = matrix([[h_elems[i][j] / h_denom[i][j] for j in range(2)] for i in range(2)])

H = h[0][0] * identity_matrix(2) + gamma
plaintext = (H.inverse() * epsilon * H) * mu * (H.inverse() * epsilon * H)
```

**关键结论：** Any commuting matrix works as the decryption key. Cayley-Hamilton guarantees that `H = c1 * I + c2 * gamma` commutes with `gamma`, and the needed scalar `c1` can be read off by comparing entries of `alpha^(-1) * gamma` against `gamma * beta`. Always check whether "private key" operations can be replaced by a commutation-equivalent derived from public data.

**参考：** TJCTF 2018 — writeup 10680

---

## BIP39 Partial-Mnemonic Brute Force via Checksum (SECCON 2018)

```python
from mnemonic import Mnemonic
lg = Mnemonic("japanese")
known = ["...23 words..."]
for w in lg.wordlist:
    try:
        if lg.check(" ".join(known + [w])):
            entropy = lg.to_entropy(" ".join(known + [w]))
            print(md5(entropy).hexdigest())
    except Exception: pass
```

**关键结论：** BIP39 has a built-in 4-bit-per-word checksum, so partial mnemonics are self-verifying. Same trick applies to Electrum's seed format and any mnemonic scheme with internal parity.

**参考：** SECCON 2018 — mnemonic, writeup 12053

---

## Asmuth-Bloom Threshold Secret Sharing via CRT (X-MAS 2018)

```python
from sympy.ntheory.modular import crt
# shares = [(s1,p1), (s2,p2), ..., (sk,pk)]
residues = [s for s, _ in shares]
moduli   = [p for _, p in shares]
S, M = crt(moduli, residues)
flag = long_to_bytes(int(S))
```

**关键结论：** Threshold sharing schemes can be CRT-based, not polynomial. Recognise Asmuth-Bloom by the `(residue, modulus)` share format; Shamir's scheme only publishes `(x, y)` coordinates without moduli.

**参考：** X-MAS CTF 2018 — writeup 12660

---

## Rabin Cryptosystem with Polynomial Primes (X-MAS 2018)

```python
from gmpy2 import iroot
# N = (r^2+3)(r^2+7) = r^4 + 10 r^2 + 21
r, _ = iroot(N - 21, 4)
p, q = r*r + 3, r*r + 7
x_p = pow(ct, (p+1)//4, p)
x_q = pow(ct, (q+1)//4, q)
# CRT combine x_p, x_q; flag is the candidate with known padding
```

**关键结论：** Any cryptosystem whose primes come from a polynomial in a small variable collapses under integer-root extraction. Recognize these by plotting `N` against hypothetical `r` guesses, or by seeing suspicious constant differences.

**参考：** X-MAS CTF 2018 — writeups 12657, 12724

---

## LCG Period Detection for Unlimited Output Prediction (X-MAS 2018)

```python
seen = {}
for i in itertools.count():
    v = fetch_next()
    if v in seen:
        period = i - seen[v]
        break
    seen[v] = i
# Now predict: future[i] == history[(i - period_start) % period]
```

**关键结论：** All LCGs are periodic and the period is bounded by `m`. Any output buffer long enough to contain the period gives you free prediction for the rest of the game.

**参考：** X-MAS CTF 2018 — writeups 12668, 12669

---

## Polynomial Coefficient Recovery via Vandermonde Linear System (X-MAS 2018)

```python
from sage.all import matrix, vector, GF
pts = [(x_i, f(x_i)) for x_i in range(degree+1)]
A = matrix([[xi**k for k in range(degree+1)] for xi, _ in pts])
b = vector([yi for _, yi in pts])
coeffs = A.solve_right(b)
```

**关键结论：** Any secret polynomial, Shamir-style sharing, or "interpolate a curve" oracle falls to a Vandermonde solve with `degree + 1` points. Sage's `solve_right` handles huge degrees.

**参考：** X-MAS CTF 2018 — writeup 12722

---

## Rabin Decryption via Four-Roots CRT Combination (Pragyan CTF 2019)

**模式（Help Rabin）：** Rabin encrypts `c = m^2 mod n` with `n = p*q` and `p, q ≡ 3 mod 4`. Once `p, q` are recovered (here by Fermat-style square-root search because `q = nextPrime(p+1)` sits right next to `p`), compute `mp = c^((p+1)/4) mod p` and `mq = c^((q+1)/4) mod q`, then combine via extended GCD to yield four square roots `±r, ±s`. Only one of the four decodes to readable text — that's the plaintext.

```python
from Crypto.Util.number import inverse

def ext_gcd(a, b):
    c0, c1, a0, a1, b0, b1 = a, b, 1, 0, 0, 1
    while c1:
        q, r = divmod(c0, c1)
        c0, c1 = c1, r
        a0, a1 = a1, a0 - q * a1
        b0, b1 = b1, b0 - q * b1
    return a0, b0, c0

# p, q already recovered (e.g. via Fermat: p ~ sqrt(n))
pe, qe = (p + 1) // 4, (q + 1) // 4
mp, mq = pow(c, pe, p), pow(c, qe, q)
yp, yq, _ = ext_gcd(p, q)                      # yp*p + yq*q == 1

r1 = (yp * p * mq + yq * q * mp) % n
r2 = n - r1
s1 = (yp * p * mq - yq * q * mp) % n
s2 = n - s1

for cand in (r1, r2, s1, s2):
    try:
        pt = bytes.fromhex(hex(cand)[2:])
        if pt.isascii(): print(pt)            # pick the readable one
    except Exception: pass
```

**关键结论：** Rabin decryption inherently produces four candidates because `x^2 ≡ c mod n` has four roots mod `n = p*q`. When `p, q ≡ 3 mod 4`, per-prime roots are the closed-form exponentiation `c^((p+1)/4) mod p` — no Tonelli-Shelanks needed. Combine with Bezout coefficients `yp*p + yq*q = 1` to get the four CRT candidates `±(yp*p*mq ± yq*q*mp) mod n`, and select by plaintext sanity (ASCII, magic bytes, known prefix). The four-root ambiguity is why Rabin typically needs redundancy in the plaintext to be useful as a cryptosystem.
