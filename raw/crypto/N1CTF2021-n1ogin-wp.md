# N1CTF 2021 - n1ogin

## 题目简述

`n1ogin` 用混合加密保护登录请求：随机生成 24 字节 AES 密钥和 24 字节 HMAC 密钥，用 RSA-PKCS#1 v1.5 加密这 48 字节密钥；业务 JSON 则使用 AES-CBC 加密，并对 `IV || ciphertext` 连续计算 7777 次 HMAC-MD5。

表面上密文同时具有机密性与完整性，但服务端解密时先检查 PKCS#7 padding，之后才验证 HMAC。两条失败路径虽然都只返回 `Error!`，耗时却相差约 110 ms，形成远程时序 padding oracle。

## 解题过程

### 确认可观测的分支差异

服务端的关键顺序如下：

```python
data = AES_CBC_decrypt(aes_key, iv, cipher)
data = unpad(data)
if not data:
    return None, "padding error"

cal_mac = iv + cipher
for _ in range(7777):
    cal_mac = HMAC_MD5(hmac_key, cal_mac)
if cal_mac != mac:
    return None, "hmac error"
```

错误 padding 会立即返回；padding 正确但密文已被篡改时，程序仍会完整执行 7777 轮 HMAC 后才失败。官方测量中，两类请求的典型耗时分别约为 70 ms 和 180 ms。HTTP/文本响应完全相同并不能消除这个 oracle，响应时间本身就是一位信息。

RSA 部分无需攻击。抓包中已有合法登录包，只需保持 `rsa_data` 不变，反复修改 `aes_data` 中 CBC 密文块并统计耗时。实际利用时对每个候选值测量多次，取中位数或较小分位数，可以削弱网络抖动造成的误判。

### 按字节恢复 CBC 明文

CBC 解密满足

$$
P_i=D_K(C_i)\oplus C_{i-1}.
$$

设中间值 $I_i=D_K(C_i)$。从一个目标块的最后一字节开始，枚举前一密文块对应字节，使修改后的明文尾部成为合法的 `01`。当响应进入较慢的 HMAC 分支时，即可确定：

$$
I_i[15]=C'_{i-1}[15]\oplus 1,qquad
P_i[15]=I_i[15]\oplus C_{i-1}[15].
$$

之后把已恢复的尾部统一调整为 `02 02`、`03 03 03`，逐字节向前推进：

```python
for pos in range(15, -1, -1):
    pad = 16 - pos
    crafted = bytearray(previous_block)

    for j in range(pos + 1, 16):
        crafted[j] = intermediate[j] ^ pad

    for guess in range(256):
        crafted[pos] = guess
        if median_elapsed(crafted, target_block) > threshold:
            intermediate[pos] = guess ^ pad
            plaintext[pos] = intermediate[pos] ^ previous_block[pos]
            break
```

对所有 CBC 块重复即可恢复完整登录 JSON，包括管理员密码。需要注意偶然出现的原始合法 padding，可通过额外翻转前一个无关字节进行二次确认。

### 登录并读取 flag

仓库的部署源码保留了恢复结果：

```text
admin password: R,YR35B7^r@'U3FV
```

用该密码重新生成一个合法登录请求，进入管理员交互界面后输入 `flag`，得到：

```text
n1ctf{R3m0t3_t1m1ng_4ttack_1s_p0ssibl3__4nd_u_sh0uld_v3r1fy_th3_MAC_f1rs7}
```

## 方法总结

问题不在 AES-CBC 或 HMAC-MD5 单独是否安全，而在认证顺序。既然协议采用 Encrypt-then-MAC，接收端就必须先以常数时间验证 MAC，验证成功后才能解密和检查 padding；否则任何可观测的错误、耗时或连接行为都可能重新暴露 padding oracle。远程时序攻击还应使用重复采样与稳健统计，而不能依赖单次延迟。
