# yqs

## 题目简述

组合密码题，融合 LWE 与椭圆曲线。先利用小误差、小维度参数对 LWE 做 primal attack 恢复私钥；该私钥又参与椭圆曲线点坐标异或，使输入点可控，从而做无效曲线小子群攻击，最后 CRT 合并恢复主密钥并 AES 解密。

## 解题过程

官方 WP 说明该题由 LLM 辅助生成，最终解法主线是 LWE primal attack 与无效曲线攻击的组合。

本题融合LWE格密码与椭圆曲线密码学双重结构。需先通过格攻击恢复LWE私钥，再利用无效曲线攻击破解ECDLP。

Phase 1: LWE Primal Attack

核心代码片段

```python
class LWE:
    def __init__(self):
        self.q = get_prime(qbits)
        self.e = (1 << ebits) - 1
        self.s = [randint(0, self.q-1) for _ in range(N)]
    def encrypt(self, m):
        m_encoded = (m & 0xFF) << ebits
        a = [randint(0, self.q-1) for _ in range(N)]
        error = randint(0, self.e)
        b = (vec_dot(a, self.s, self.q) + m_encoded + error) % self.q
        return (self._vec_to_int(a), int(b))
```

分析：

LWE 问题标准形式：给定矩阵A和向量b = A·s + e + m，求秘密向量s。本题参数较小(q≈2^77, N=77,

e∈[0,3])，适合直接primal attack。构造格矩阵将误差作为短向量目标，LLL规约后可恢复私钥。AI助手提示此处需控制采样数量在100组以内避免噪声累积。

Phase 2: Invalid Curve Attack

核心代码片段

```python
def vec2key(v, q, N):
    x = sum(int(v[i]) * (q**i) for i in range(N))
    return int.from_bytes(sha256(str(x).encode()).digest(), 'little')
priv = lwe.export_priv_key()
x ^= priv
y ^= priv
result = ecc.mul((x, y), master_sec)
```

分析：

交换接口存在设计缺陷：未验证输入点是否在曲线上。通过XOR操作可任意控制最终传入标量乘法的点坐标。选择小阶曲线上的点，可将ECDLP分解至小阶子群求解。AI分析表明需收集约242位CRT模数方可恢复主密钥。

Phase 3: ECDLP in Small Subgroups

核心代码片段

```python
def find_invalid_curve_point(limit_bits=40):
    while True:
        b = random element
        E = EllipticCurve(GF(P), [A, int(b)])
        order_E = E.order()
        factors = factor(order_E)
        for pr, exp in factors:
            if pe.nbits() <= limit_bits:
                Q = exact_order_point(E, pe)
                if Q: return {"curve_b": b, "point": Q, "order": pe}
```

分析：

无效曲线攻击核心在于寻找具有小素因子的曲线阶。对每个候选曲线，计算其阶并分解，筛选出含有小于40位素因子的曲线。在该子群中求解DLP可直接使用Pohlig-Hellman或暴力枚举。多轮CRT组合即可恢复完整主密钥。

### 完整 Exploit

```python
from sage.all import *
from pwn import *
from hashlib import sha512, sha256
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
from util import N, vec_dot, vec_add, vec_scalar_mul
P = 2**255 - 19
A = 486662
B = 35039673
GXY = (9,
14781619447589544791020593568409986887264606134616475288964881837755586237401)
MAX_MSG = 100
SMALL_FACTOR_LIMIT = 40
TARGET_BITS = 242
def pack_point(point):
    x, y = point
    return (int(x) << 256) | int(y)
def unpack_point(value):
    x = value >> 256
    y = value & ((1 << 256) - 1)
    return x, y
def int_to_vec(x, q, n=N):
    out = []
    for _ in range(n):
        out.append(x % q)
        x //= q
    return out
def vec_to_int(v, q):
    out = 0
    for i, coeff in enumerate(v):
        out += int(coeff) * (q**i)
    return out
def forge_input_coord(priv, target_coord):
    low_mask = (1 << 256) - 1
    priv_low = priv & low_mask
    priv_high = priv >> 256
    offset = int((Integer(priv_high) * (1 << 256)) % P)
    wanted_low = int((target_coord - offset) % P)
    return wanted_low ^ priv_low
def primal_attack2(A_mat, b_vec, m, n, p, esz):
    L = block_matrix([
        [matrix(Zmod(p), A_mat).T.echelon_form().change_ring(ZZ), 0],
        [matrix.zero(m - n, n).augment(matrix.identity(m - n) * p), 0],
        [matrix(ZZ, b_vec), 1],
    ])
    Q = diagonal_matrix([1] * m + [esz])
    L *= Q
    L = L.LLL()
    L /= Q
    res = L[0]
    if res[-1] == 1:
        e = vector(GF(p), res[:m])
    elif res[-1] == -1:
        e = -vector(GF(p), res[:m])
    else:
        return None
    s = matrix(Zmod(p), A_mat).solve_right(vector(Zmod(p), b_vec) - e)
    return vector(GF(p), list(s))
class LocalChall:
    def __init__(self):
        self.io = process(['python', 'task.py'])
        self.pub = None
        self.enc = None
        self.round_idx = 0
        self.q = None
        self.started = False
    def read_banner_and_round(self):
        if not self.started:
            self.pub = int(self._parse_after_colon(self.io.recvline()))
            self.enc = self._parse_after_colon(self.io.recvline())
            self.started = True
        self.q = int(self._parse_after_colon(self.io.recvline()))
        self.round_idx += 1
        self.io.recvuntil(b"Channel your intent (m=message, e=exchange):")
        return self.pub, self.enc, self.q
    def _parse_after_colon(self, line):
        return line.decode().strip().split(": ", 1)[1]
    def send_m(self, msg):
        self.io.sendline(b"m")
        self.io.recvuntil(b"Offer your spiritual essence (int):")
        self.io.sendline(str(msg).encode())
        line = self.io.recvline()
        ct = eval(self._parse_after_colon(line))
        self.io.recvuntil(b"Channel your intent (m=message, e=exchange):")
        return ct
    def send_e(self, packed):
        self.io.sendline(b"e")
        self.io.recvuntil(b"Forge your spirit formation (int):")
        self.io.sendline(str(packed).encode())
        line = self.io.recvline()
        value = int(self._parse_after_colon(line))
        return value
    def close(self):
        self.io.close()
def recover_round_secret(io, q):
    rows = []
    rhs = []
    for msg in range(MAX_MSG):
        m8 = int(sha512(str(msg).encode()).hexdigest(), 16) & 0xFF
        a_int, b_val = io.send_m(msg)
        rows.append(int_to_vec(a_int, q))
        rhs.append((b_val - (m8 << 2)) % q)
    A_mat = matrix(GF(q), rows)
    b_vec = vector(GF(q), rhs)
    s = primal_attack2(A_mat, b_vec, MAX_MSG, N, q, 4)
    if s is None:
        raise ValueError("LWE recovery failed")
    priv = vec_to_int(s, q)
    return s, priv
def curve_b_from_point(x, y):
    return int((y * y - x * x * x - A * x) % P)
def exact_order_point(E, order_target, trials=64):
    order_E = E.order()
    if order_E % order_target != 0:
        return None
    cofactor = order_E // order_target
    for _ in range(trials):
        Q = cofactor * E.random_point()
        if Q.is_zero():
            continue
        if Q.order() == order_target:
            return Q
    return None
def find_invalid_curve_point(limit_bits=SMALL_FACTOR_LIMIT):
    gf = GF(P)
    used_b = set([B])
    while True:
        b = ZZ.random_element(1, P - 1)
        if b in used_b:
            continue
        used_b.add(int(b))
        if (4 * A**3 + 27 * b**2) % P == 0:
            continue
        E = EllipticCurve(gf, [A, int(b)])
        order_E = E.order()
        factors = factor(order_E)
        small = []
        for pr, exp in factors:
            pe = int(pr**exp)
            if Integer(pe).nbits() <= limit_bits:
                small.append(pe)
        small.sort(reverse=True)
        for order_target in small:
            Q = exact_order_point(E, order_target)
            if Q is not None:
                return {
                    "curve_b": int(b),
                    "curve_order": int(order_E),
                    "point": (int(Q[0]), int(Q[1])),
                    "order": int(order_target),
                }
def build_candidate_pool(target_bits=TARGET_BITS,
limit_bits=SMALL_FACTOR_LIMIT):
    modulus = 1
    pool = []
    while Integer(modulus).nbits() < target_bits:
        candidate = find_invalid_curve_point(limit_bits=limit_bits)
        r = candidate["order"]
        if gcd(modulus, r) != 1:
            continue
        pool.append(candidate)
        modulus *= r
    return pool
def residue_from_response(candidate, response_int):
    x, y = unpack_point(response_int)
    E = EllipticCurve(GF(P), [A, candidate["curve_b"]])
    P0 = E(*candidate["point"])
    Q0 = E(x, y)
    return int(discrete_log(Q0, P0, ord=candidate["order"], operation="+"))
def one_round_invalid_curve_residue(io, candidate):
    _, _, q = io.read_banner_and_round()
    _, priv = recover_round_secret(io, q)
    px, py = candidate["point"]
    in_x = forge_input_coord(priv, px)
    in_y = forge_input_coord(priv, py)
    masked = pack_point((in_x, in_y))
    response = io.send_e(masked)
    if response == 0:
        raise ValueError("point at infinity")
    residue = residue_from_response(candidate, response)
    return {"q": q, "priv": priv, "response": response, "residue": residue}
def recover_master_secret_locally(candidates):
    io = LocalChall()
    logs = []
    moduli = []
    try:
        for idx, candidate in enumerate(candidates, 1):
            result = one_round_invalid_curve_residue(io, candidate)
            pub_int = io.pub
            enc_hex = io.enc
            logs.append(result["residue"])
            moduli.append(candidate["order"])
            d = int(crt(logs, moduli))
            crt_mod = prod(moduli)
            E = EllipticCurve(GF(P), [A, B])
            G = E(*GXY)
            pub = E(*unpack_point(pub_int))
            if d * G == pub:
                return d, pub_int, enc_hex
        return int(crt(logs, moduli)), pub_int, enc_hex
    finally:
        io.close()
def decrypt_flag(d, enc_hex):
    key = sha256(str(d).encode()).digest()
    ct = bytes.fromhex(enc_hex)
    pt = AES.new(key, AES.MODE_ECB).decrypt(ct)
    try:
        pt = unpad(pt, 32)
    except ValueError:
        pass
    return pt
def main():
    context.log_level = "error"
    candidates = build_candidate_pool()
    d, pub_int, enc_hex = recover_master_secret_locally(candidates)
    pt = decrypt_flag(d, enc_hex)
    print(pt.decode() if isinstance(pt, bytes) else pt)
if __name__ == "__main__":
    main()
```

## 方法总结

- 核心技巧：LWE primal attack + ECC 无效曲线小子群 + CRT 恢复标量。
- 识别信号：服务端没有验证输入点是否在曲线上，且点坐标还被可恢复秘密异或时，应考虑 invalid curve attack。
- 复用要点：先恢复用于坐标变换的中间秘密，再挑选小阶无效曲线点分轮拿 residue，累计足够 CRT 模数后验证公钥。
