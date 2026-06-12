# ZJUAM Just Uses Awful Math

## 题目简述

附件是 ZJUAM 登录流程的明文 HTTP 抓包。登录页会从 `/cas/v2/getPubKey` 获取 RSA 公钥，再由前端 JavaScript 把密码反转后做无填充 RSA 加密。题目里的 RSA 模数只有 256 bit，可以直接分解，因此从抓包中恢复公钥和密文后即可解密出登录前的明文。

## 解题过程

用 Wireshark 或 `tcpdump` 直接查看 HTTP 流量：

```bash
tcpdump -A -s 0 -nn -r zjuam.pcap
```

关键请求链为：

```text
GET  /cas/login
GET  /cas/v2/getPubKey
POST /cas/login
```

`/cas/v2/getPubKey` 返回 RSA 参数：

```json
{
  "modulus": "90011418f37a7a075aead75a9829d38eb2d750fd17bb24e5861b89d7658a88c3",
  "exponent": "10001"
}
```

登录表单中可以看到用户名和密文：

```text
username=player
password=590948ad2f7a3c0b1a2a5e5f470f4297db3b90623251132be2c5e5395cd12563
```

登录页引用的前端脚本中，加密逻辑可以整理为：

```javascript
var key = new RSAUtils.getKeyPair(public_exponent, "", Modulus);
var reversedPwd = password.split("").reverse().join("");
var encrypedPwd = RSAUtils.encryptedString(key, reversedPwd);
```

也就是先反转密码字符串，再进行 textbook RSA 加密，没有 OAEP/PKCS#1 padding。`security.js` 内部还用 16-bit little-endian digit 存储大整数，每个 digit 存两个字符，因此解密后需要按这个编码方式还原字节。

模数为：

$$
n=\texttt{0x90011418f37a7a075aead75a9829d38eb2d750fd17bb24e5861b89d7658a88c3}
$$

只有 256 bit。分解得到：

```text
p = 202555251191383333988748320354737959551
q = 321566364572398185024295275472079273917
```

验证 $p\cdot q=n$ 后计算私钥：

$$
\varphi(n)=(p-1)(q-1),\qquad d=e^{-1}\bmod\varphi(n)
$$

然后执行：

$$
m=c^d\bmod n
$$

按前端 16-bit little-endian digit 方式还原明文，再把字符串反转回来：

```python
def js_bigint_to_bytes(value):
    out = bytearray()
    while value:
        digit = value & 0xffff
        out.append(digit & 0xff)
        out.append(digit >> 8)
        value >>= 16
    return bytes(out).rstrip(b"\x00")

n = int("90011418f37a7a075aead75a9829d38eb2d750fd17bb24e5861b89d7658a88c3", 16)
e = 0x10001
c = int("590948ad2f7a3c0b1a2a5e5f470f4297db3b90623251132be2c5e5395cd12563", 16)
p = 202555251191383333988748320354737959551
q = 321566364572398185024295275472079273917

assert p * q == n
d = pow(e, -1, (p - 1) * (q - 1))
plain_reversed = js_bigint_to_bytes(pow(c, d, n)).decode()
print(plain_reversed[::-1])
```

解密得到的反转字符串为：

```text
}dLR0w_EHT_sev@s_SLT{FTCA
```

反转后得到：

```text
ACTF{TLS_s@ves_THE_w0RLd}
```

脚本最终输出 `ACTF{...}`，验证了数学约束求解结果。

## 方法总结

- 核心技巧：从明文 HTTP 抓包中恢复前端 RSA 参数，分解弱模数后按前端编码方式解密。
- 识别信号：统一认证流量可见、前端自实现 RSA、模数只有 256 bit、无 padding，都是 textbook RSA 可直接攻击的信号。
- 复用要点：抓包题要把请求字段、前端加密代码、模数分解和解密后编码还原串起来；仅有 $m=c^d\bmod n$ 还不够，还要还原 JS 大整数编码和业务层反转逻辑。
