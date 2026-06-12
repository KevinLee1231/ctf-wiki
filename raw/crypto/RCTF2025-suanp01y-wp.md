# SuanP01y

## 题目简述

题目给出稀疏多项式构造的提示，核心关系可看成 `h1 * h0^{-1}` 在 `GF(2)[X]/(X^r-1)` 中的像。利用有界次数做 rational reconstruction 可恢复分母多项式，再枚举循环位移得到 AES-CTR 密钥。

## 解题过程

### 关键观察

题目给出稀疏多项式构造的提示，核心关系可看成 `h1 * h0^{-1}` 在 `GF(2)[X]/(X^r-1)` 中的像。

### 求解步骤

h0 and h1 are sparse polynomials built by summing 41 monomials drawn from indices ≤
⌊16381/3⌋ and then rotated by random exponents. Since h1 and h0 are coprime and we know
their quotient modulo X^r−1, we can treat the hint polynomial as the image of h1 * h0^{-1} in
GF(2)[X]/(X^r−1) and run rational reconstruction to recover h0 (up to a cyclic rotation).
from sage.all import *
from Crypto.Cipher import AES
from hashlib import md5
import sys

r = 16381
R = PolynomialRing(GF(2), 'x')
x = R.gen()
mod_poly = x**r - 1

def parse_polynomial(poly_str):
    poly = R.zero()
    for term in poly_str.replace(' ', '').split('+'):
        if not term:
            continue
        if term == '1':
            poly += 1
        elif term == 'X':
            poly += x
        elif term.startswith('X^'):
            exponent = int(term[2:])
            poly += x**exponent
        else:
            raise ValueError(f"Unknown term: {term}")
    return poly

def format_poly(exps):
    terms = []
    for e in sorted(exps, reverse=True):
        if e == 0:
            terms.append('1')
        elif e == 1:
            terms.append('X')
        else:
            terms.append(f'X^{e}')
    return ' + '.join(terms) if terms else '0'

def recover_denominator(hint_poly):
    den_bound = r // 3
    num_bound = r - den_bound
    num, den = hint_poly.rational_reconstruction(mod_poly, num_bound,
den_bound)
    den_exps = sorted(m.degree() for m in den.monomials())
    return den_exps

def brute_force_flag(den_exps, ciphertext_hex):
    ciphertext = bytes.fromhex(ciphertext_hex.strip())
    nonce = b'suanp01y'
    for shift in range(r):
        shifted = [ (e + shift) % r for e in den_exps ]
        poly_str = format_poly(shifted)
        key = md5(poly_str.encode()).digest()
        cipher = AES.new(key, AES.MODE_CTR, nonce=nonce)
        plaintext = cipher.decrypt(ciphertext)
        if plaintext.startswith(b'RCTF{'):
            print(f'Shift: {shift}')
            print(plaintext.decode())
            return plaintext.decode()
    raise ValueError('Flag not found')

def main(path):
    with open(path, 'r') as f:
        lines = [line.strip() for line in f if line.strip()]
    if len(lines) < 2 or not lines[0].startswith('hint = '):
        raise ValueError('Unexpected file format')
    hint_str = lines[0].split('hint = ', 1)[1]

### 跨页补回：多项式恢复脚本收尾

myexample.py
    ciphertext_hex = lines[1]
    hint_poly = parse_polynomial(hint_str)
    den_exps = recover_denominator(hint_poly)
    brute_force_flag(den_exps, ciphertext_hex)

if __name__ == '__main__':
    path = sys.argv[1] if len(sys.argv) > 1 else 'output.txt'
    main(path)

## 方法总结

- 核心技巧：多项式有理重构恢复稀疏分母。
- 识别信号：已知商多项式、分子/分母次数有界且在循环商环中互素。
- 复用要点：恢复结果常有旋转等价类，需要枚举位移并用 flag 格式验证。
