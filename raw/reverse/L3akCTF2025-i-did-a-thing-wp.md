# L3akCTF 2025 I Did a Thing Writeup

## 题目简述

题目附件 `chal.zip` 中包含经过混淆的 JavaScript 和 `cipher.txt`。程序使用自定义虚拟机隐藏密钥派生逻辑，再用动态 S 盒、MDS 矩阵和轮密钥组成的小型 SPN 加密 flag。

仓库保留了两个版本：比赛实际下发的 `ctfd-dist` 意外关闭了大部分 opcode shuffle，只执行了两次打乱；`ideal-version-dist` 则是作者原本想发布的完整版本。两者使用不同的 VM 派生常量和时间戳，但最终 flag 相同。决定性障碍是恢复混淆程序的行为并逆转自定义密码流程，因此按 Reverse 归档。

## 解题过程

### 从虚拟机中恢复派生材料

JavaScript 外层使用 JSFuck 和自定义字节码解释器隐藏了两段源码：一个 RC4 实现和 VM 解释器本身。可以在动态执行时拦截 `eval`/`Function` 的参数，或绕过完整性检查后转储被执行的字符串。随后按程序实现的 Murmur 风格 32 位哈希分别计算两段源码的值，再将十进制结果直接拼接。

两个附件对应的 `info` 材料分别为：

```text
比赛版本：28603234533519191490
理想版本：18717984172070616427
```

程序中还存在一个明确的诱饵：

```text
L3AK{y0u_w1sh_7h15_w4s_th3_fl46_y0u_w3r3_l00k1ng_f0r}
```

它不是 flag，但其前 16 字节用作 HKDF 的 salt，剩余部分参与后续 HMAC 掩码。不能因为字符串具有合法 flag 格式就提前停止分析。

### 从 ZIP 时间恢复动态参数

加密时使用 `cipher.txt` 的 ZIP 条目时间。把候选 Unix 时间戳按大端序编码为 8 字节，并用 HKDF-SHA256 派生：

```python
t_bytes = struct.pack(">Q", timestamp)
prk = hmac.new(salt, t_bytes, hashlib.sha256).digest()
okm = hkdf_expand(prk, info, 256 + 16 + 4 * 10)
```

派生输出依次用于：

1. 前 256 字节构造动态 S 盒及逆 S 盒；
2. 接下来的 16 字节构成 $4\times4$ MDS 矩阵，并在 $GF(2^8)$ 上求逆；
3. 最后 40 字节组成 10 组 4 字节轮密钥。

ZIP 的 DOS 时间字段只有年月日时分秒，没有时区。若直接对 `ZipInfo.date_time` 调用本机时区相关的 `timestamp()`，官方脚本的 $\pm120$ 秒窗口可能完全错位。可把归档时间视为无时区墙钟，枚举 $-14$ 到 $+14$ 小时的时区偏移，再在每个候选附近枚举两分钟。

在 Asia/Shanghai 环境中复现时，两个附件都需要相对直接解释结果校正 12 小时，得到：

```text
比赛版本时间戳：1749504275
理想版本时间戳：1752447350
```

正确时间戳还必须让 MDS 可逆，并使最终明文通过填充和 UTF-8 检查，因此候选很容易确认。

### 解除掩码并逆转 CBC-SPN

先计算嵌入密文的 8 字节标签：

```python
tag = hmac.new(
    t_bytes,
    not_the_flag[16:].encode(),
    hashlib.sha256,
).digest()[:8]

pos = (
    sum(map(sum, mds)) +
    sum(map(sum, round_keys))
) % len(ciphertext)

for i, b in enumerate(tag):
    ciphertext[(pos + i) % len(ciphertext)] ^= b
```

密文以 4 字节为一组，第一组是 CBC IV。对其余各组逆向执行 10 轮：

```python
def decrypt_block(block, inv_sbox, inv_mds, round_keys):
    state = bytes(block)
    for key in reversed(round_keys):
        state = bytes(a ^ b for a, b in zip(state, key))
        state = gf_matrix_multiply(inv_mds, state)
        state = bytes(inv_sbox[b] for b in state)
    return state
```

每个解密块再与前一个密文块异或，最后检查并删除 PKCS#7 填充。比赛版和理想版均解出：

```text
L3AK{V1rtu4l_M4ch1n3_M4st3r_H00r4y!!!:)}
```

## 方法总结

本题的难点不在某个标准密码算法，而在把 VM 混淆、源码完整性哈希、ZIP 元数据、HKDF 和自定义 SPN 串成一条准确的数据流。适合先动态截获解释器真正执行的源码，再按派生顺序逐层复现，避免在 JSFuck 文本中做无意义的人工阅读。

复现时必须特别处理无时区的 ZIP 时间。归档时间只保存墙钟值，本机时区不同会造成整小时级偏移，远超官方脚本的两分钟容差。正确做法是显式枚举合理时区偏移，并用 MDS 可逆性、合法填充和 flag 格式共同验证候选。仓库中的两个版本也应分别说明，不能把赛后修正版本误写成比赛时唯一附件。
