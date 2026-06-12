# OhMyCaptcha

## 题目简述

题目把“验证码校验”和 Python 表达式沙箱组合在一起。验证码不是图像识别，而是对提交整数 `message` 的模运算约束；通过校验后，`long_to_bytes(message)` 会进入 `eval()`，输出再被无 padding RSA 以固定指数 $e=5$ 加密。服务端每轮还生成 AES-CTR 的 `key`、`nonce` 和 `cipher`，flag 需要从这些变量关系中恢复。

## 解题过程

附件 `task.py` 的验证码逻辑为：

```python
self.code = ''.join(random.sample("0123456789", 10))
...
return 0 <= message < prod(self.pn) and all(str(message % p) in self.code for p in self.pn)
```

`random.sample("0123456789", 10)` 每次都会取出全部十个数字，只是顺序随机。因此只要构造 `message`，让它对每个手机号素数 $p_i$ 的余数都落在 `0..9`，就必然通过验证码。

通过校验后执行：

```python
sandbox = SandboxService(getPrime(0x137), getPrime(0x137))
key, nonce = [os.urandom(8).hex() for _ in ":)"]
cipher = AES.new(key.encode(), AES.MODE_CTR, nonce=bytes.fromhex(nonce)).encrypt(FLAG).hex()

template = (
    f"key = {key!r}\n"
    f"cipher_with_key_{key}_{nonce} = {cipher!r}\n"
    f"print(eval({long_to_bytes(message)}))"
)
print(sandbox.run(template).hex())
```

也就是说，每个合法 `message` 实际是一段 Python 表达式，表达式输出会被 RSA 加密：

$$
c=m^5\bmod n
$$

验证码构造使用 CRT。令：

$$
P=\prod_i p_i,\qquad M_i=\frac{P}{p_i},\qquad N_i=M_i^{-1}\bmod p_i
$$

若希望：

$$
message\bmod p_i=a_i,\qquad a_i\in[0,10]
$$

则：

$$
message=\sum_i a_iM_iN_i\bmod P
$$

选择 64 个 11 位手机号素数，并把 `message` 的高位约束为目标命令前缀。为了让余数集中在小范围内，先加一个 bias，相当于让系数围绕 5 平衡：

```python
bias = 0
coeffs = []
for p in ps:
    coeff = pow(N // p, -1, p) * (N // p) % N
    bias += (-5) * coeff % N
    coeffs.append(coeff)
```

之后构造 LLL 格，找到高位以命令开头、同时所有余数都在 `0..10` 的 `message`。需要构造四条命令：

```python
COMMANDS = [
    b"key#",
    b"[*vars()][8]",
    b"vars()",
    b"vars()[[*vars()][8]]",
]
```

这些命令分别用于：

- `key#`：输出 AES key；
- `[*vars()][8]`：输出变量名 `cipher_with_key_<key>_<nonce>`，用于恢复 nonce；
- `vars()`：输出整个变量字典；
- `vars()[[*vars()][8]]`：输出 `cipher` 字符串。

因为服务端给验证码时会输出一个包含全部数字的随机排列，脚本会反复连接，直到验证码字符串中包含 `"10"`，确保余数上界 `10` 的构造也能通过字符串包含校验。

拿到四个 RSA 输出后，先恢复 key。`key` 是 16 个 hex 字符，`key#` 中的 `#` 会注释掉后续不可控字节，输出明文较短。由于 $e=5$ 且明文小，可以枚举：

$$
r_1+i n
$$

并检查是否为精确五次方：

```python
for i in range(100000):
    tmp = r1 + i * n
    key, exact = ZZ(tmp).nth_root(5, truncate_mode=True)
    if exact:
        key = long_to_bytes(key).decode()
        break
```

接着恢复 nonce。变量名形如：

```text
cipher_with_key_<key>_<nonce>
```

nonce 是 16 个 hex 字符。已知 key 后，枚举 nonce 前两个 hex 字符，剩余 14 字节作为小根，用 Coppersmith 解：

```python
m_high = bytes_to_long(f"cipher_with_key_{key}_{i}{j}".encode()) << (14 * 8)
P.<x> = PolynomialRing(Zmod(n))
f = (x + m_high)^5 - r2
root = f.monic().small_roots(X=2^(14*8-1), beta=0.4, epsilon=0.005)
```

第三步用 Franklin-Reiter 恢复 `cipher % n`。`vars()` 和 `vars()[[*vars()][8]]` 的输出之间存在仿射关系，写成：

$$
m_3=a x+b,\qquad m_4=x
$$

其中 $a=256^2$，$b$ 是固定模板前缀：

```python
template = (
    "{'__name__': '__main__', ... "
    f"'key': '{key}', 'cipher_with_key_{key}_{nonce}': '{tmp}'}}"
)
a = 256**2
b = bytes_to_long(template.encode())
```

于是对：

$$
g_1(x)=x^5-r_4,\qquad g_2(x)=(ax+b)^5-r_3
$$

求多项式 gcd，即可得到 `cipher % n`。

最后恢复完整 hex cipher。cipher 是十六进制字符串，脚本把每个 hex 字符表示为：

```text
ord(hex_char) = 48 * x + y + 52
```

其中 `x,y` 是小范围整数。构造关于 94 个 hex 字符的格，使：

$$
\sum_i (48x_i+y_i+52)\cdot256^{k-1-i}\equiv cipher\bmod n
$$

LLL/BKZ 后恢复整段 hex cipher。拿到 `key`、`nonce` 和 `cipher` 后，按 AES-CTR 解密：

```python
flag = AES.new(
    key.encode(),
    AES.MODE_CTR,
    nonce=bytes.fromhex(nonce),
).decrypt(bytes.fromhex(cipher))
```

最终得到：

```text
ACTF{Oo0oo00OoO00O0Oo0oh_My_C4p7cHa_WoR1d!!!!!}
```

批量求解脚本完成多轮 captcha/格规约恢复后输出 flag，说明参数恢复流程正确。

## 方法总结

- 核心技巧：用 CRT+LLL 构造可通过验证码的 Python 表达式，再利用 $e=5$ RSA 的代数关系恢复 AES-CTR 参数和密文。
- 识别信号：验证码字符集合恒定、`eval(long_to_bytes(message))`、无 padding RSA 和同一模板变量同时出现时，应把表达式输出设计成代数 oracle。
- 复用要点：先构造短命令泄漏 key/变量名，再用 Coppersmith、Franklin-Reiter 和 hex 字符格逐步恢复 nonce、`cipher % n` 与完整 cipher。
