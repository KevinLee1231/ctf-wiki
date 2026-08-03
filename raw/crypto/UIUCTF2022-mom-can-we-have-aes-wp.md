# mom can we have AES

## 题目简述

题目模拟了一次自制的 TLS 握手：客户端与服务端分别监听一个端口，协商 AES 模式，服务端用 RSA 签名固定证书并解密 premaster secret，双方再从 `client_random || server_random || premaster_secret` 派生会话密钥。握手结束后，客户端会返回

$$
E_K(\operatorname{pad}(\text{attacker-prefix}\parallel\text{flag})).
$$

握手没有把协商出的密码套件绑定进签名或完整性校验。攻击者可以居中转发证书、随机数和密钥交换数据，同时把双方的可选套件都压缩为 `AES.MODE_ECB`。随后客户端就变成固定密钥下的可控前缀 ECB oracle。

## 解题过程

### 降级密码套件

客户端提供 CBC、CTR、OFB、EAX 和 ECB，服务端提供 CBC、CTR、EAX、GCM 和 ECB。攻击者分别连接两端，只向服务端声称客户端支持 ECB，再只向客户端声称服务端选择 ECB：

```text
client  <--- signed certificate / random / key exchange --->  attacker  <--- relay --->  server
client  <-------------------- "AES.MODE_ECB" -------------------------->  server
```

服务端证书签名仍然有效，因为攻击者原样转发签名；RSA-OAEP 加密的 premaster secret 也无需解密，只需转发给服务端。问题在于认证对象只有固定字符串 `SIGPwny`，并不覆盖协商 transcript，所以任何一端都无法发现套件被修改。

完成转发后，双方得到同一会话密钥且都使用 ECB。攻击者继续转发加密的 `finish`，再向客户端的通信循环提交任意十六进制前缀。

### ECB 字节级恢复

先发送空前缀记录基准密文长度，再逐渐增加填充字节。密文第一次增长时，可以推出填充边界和 flag 长度。尽管题目把 PKCS#7 的 `block_size` 设置为 32，底层 AES 的比较单位仍是 16 字节块；实际攻击可以直接从相邻密文块是否相同来确定边界。

恢复第 $i$ 个字节时，构造前缀，使未知字节落在当前目标块末尾。先查询目标密文，再枚举可打印字节 $b$，查询

$$
\text{padding}\parallel\text{known}\parallel b\parallel\text{flag},
$$

若相关 ECB 块与目标相同，则 $b$ 就是下一个 flag 字节。官方 solver 的比较函数会随着已恢复长度增加而比较更多前缀块，从而跨块继续恢复：

```python
known = b""
for i in range(flag_length):
    align = b" " * (16 - 1 - (i % 16))
    target = oracle(align)
    block = i // 16

    for candidate in range(32, 128):
        probe = align + known + bytes([candidate])
        out = oracle(probe)
        lo, hi = block * 16, (block + 1) * 16
        if out[lo:hi] == target[lo:hi]:
            known += bytes([candidate])
            break
```

这里的 `oracle()` 表示向客户端发送十六进制前缀并解析其返回的密文；代码是攻击核心的等价写法，连接、PoW 和握手转发部分可直接按官方 `healthcheck/solve.py` 实现。

最终逐字节恢复出：

```text
uiuctf{AES_@_h0m3_b3_l1ke3}
```

## 方法总结

- 核心技巧：利用未绑定 transcript 的套件协商实施 ECB 降级，再对 `prefix || secret` 加密 oracle 做字节级恢复。
- 识别信号：签名只覆盖固定证书、密码套件由未认证明文决定、攻击者可同时连接协议两端，以及固定密钥 ECB 加密可控前缀与秘密后缀。
- 复用要点：中间人不一定需要知道会话密钥；能够一致地修改协商结果并转发其余握手即可。攻击前还要区分协议填充粒度和底层分组密码的真实 16 字节块大小。
