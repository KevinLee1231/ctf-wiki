# Bytecoin

## 题目简述

服务用 ChaCha20-Poly1305 加密 32 字节 flag，并再以另一把 32 字节密钥对 `ciphertext || poly1305_tag` 计算 HMAC-SHA256。用户可提交密文、IV、Poly1305 标签和 HMAC 标签请求解密。题目结合了两个实现错误：十六进制解析器会泄露复用栈缓冲区中的 HMAC 密钥；程序又忽略 ChaCha20-Poly1305 解密函数的失败返回值，仍使用认证失败后写入的明文缓冲区。

## 解题过程

每轮开始时，程序先把全局 `hmacKey` 复制到局部 `buffer[32]`，随后复用同一缓冲区接收密文。解析函数在验证两个字符是否为合法十六进制之前就执行 `n_read++`：

```c
n_read++;
if (sscanf(&buf[2*i], "%2x", &val) == 1)
    result[i] = (unsigned char)val;
else
    break;
```

若输入以 `..` 等非法十六进制对结尾，该位置不会覆盖 `buffer`，但返回长度已经把它计入。随后 `memcpy(ciphertext, buffer, messageLen)` 和打印逻辑会把未覆盖的旧字节一并回显。服务总共执行 33 轮，因此可用 32、31、……、1 字节的输入逐轮泄露密钥尾部：

```python
key = [b""] * 32
for i in range(1, 33):
    # 32-i 个零字节，末尾追加一对非法字符
    p.sendline(b"00" * (32 - i) + b"..")
    # 其余 IV/tag 输入按要求补足；读取“Decrypting message”行
    leaked_hex = get_printed_ciphertext()
    key[-i] = leaked_hex[-2:]

hmac_key = bytes.fromhex(b"".join(key).decode())
```

可先对本轮服务器给出的原密文与 Poly1305 标签重新计算 HMAC，确认恢复的密钥无误。

ChaCha20 是流密码。把密文前三字节改掉，就会可预测地改变明文前三字节，从而绕过程序对 `plaintext[0:3] == b"byu"` 的拒绝检查。此时 Poly1305 标签必然失效，但程序只调用：

```c
result = wc_ChaCha20Poly1305_Decrypt(..., plaintext);
/* result 从未检查 */
```

在题目所用 WolfSSL 行为下，即使函数返回认证失败，`plaintext` 仍包含解密结果。攻击者可令 Poly1305 标签全零，并用已恢复的 HMAC 密钥为修改后的 `ciphertext || zero_tag` 生成合法 HMAC：

```python
modified_ct = b"\x00\x00\x00" + original_ct[3:]
fake_poly = b"\x00" * 16
fake_hmac = HMAC.new(
    hmac_key,
    msg=modified_ct + fake_poly,
    digestmod=SHA256,
).digest()
```

程序通过外层 HMAC 后输出认证失败产生的明文。前三字节已损坏，但原 flag 前缀已知为 `byu`，替换回来即可得到：

```text
byuctf{crypt0_buffer_reuse_b4d}
```

## 方法总结

- 核心技巧：利用解析长度与实际写入长度不一致泄露栈上旧密钥，再利用未检查 AEAD 返回值取得未认证明文。
- 识别信号：敏感密钥与用户输入复用同一缓冲区、计数在解析成功前递增、认证解密返回值被丢弃，都是强烈危险信号。
- 复用要点：HMAC 本身没有被破解；先泄露 HMAC 密钥才能伪造外层校验。任何 AEAD API 返回失败后都必须立即丢弃并清零输出缓冲区，不能继续检查或打印明文。
