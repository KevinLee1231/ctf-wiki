# BTS

## 题目简述

服务实现了自制的 ChaCha-HMAC token。用户提交 JSON 后，服务拒绝 `name == "admin"`，再用随机 nonce、固定密钥生成 token；提交 token 时会先验证 HMAC，再解密并重新解析 JSON。漏洞不是 ChaCha 或 HMAC 本身，而是加解密两端对 `zip` 参数顺序使用不一致。

加密端执行：

```python
r = bytes(x ^ y for x, y in zip(plaintext, keystream))
```

解密端却执行：

```python
ciphertext_iterator = iter(ciphertext)
r = bytes(x ^ y for x, y in zip(keystream, ciphertext_iterator))
```

由于 Python 的 `zip` 在最短迭代器耗尽前，可能已经从左侧迭代器额外取走一个元素，跨 64 字节块时会造成密钥流错位。

## 解题过程

构造长度超过一个 ChaCha 块的 JSON，并让第二块开头的一个字节承担错位。官方 solver 使用：

```python
payload = b'{"name":'.ljust(63) + b'" admin"}'
```

该字符串总长 72 字节，服务第一次解析出的对象是：

```python
{"name": " admin"}
```

因为值前有一个空格，所以不会触发 `data["name"] == "admin"` 的拒绝逻辑。加密第一轮中，`zip(plaintext, keystream)` 为确认 64 字节密钥流已耗尽，会额外从明文迭代器取走第 65 个字节，但该字节不会进入输出；下一轮从第 66 个明文字节开始。

解密时参数顺序反过来，左侧密钥流是固定 64 字节序列，右侧密文迭代器不会被额外消耗。最终恢复的明文因此少了原始第 65 个字节，也就是字符串值开头用于绕过检查的空格，解密后 JSON 变成：

```python
{"name": "admin"}
```

完整交互只需申请该 token，再原样提交：

```python
from pwn import remote

io = remote("HOST", 36000)
payload = b'{"name":'.ljust(63) + b'" admin"}'

io.sendlineafter(b"choice", b"1")
io.sendlineafter(b"input", payload)
io.recvuntil(b"Token:")
token = io.recvline().strip()

io.sendlineafter(b"choice", b"2")
io.sendlineafter(b"token", token)
io.interactive()
```

HMAC 仍然有效，因为 token 没有被修改；改变发生在服务自身错误的解密迭代过程中。最终得到：

```text
grey{D0N7_r011_Y0Ur_0WN_CrYP70_feae2400bc86b21c}
```

## 方法总结

- 核心技巧：利用 Python `zip` 的迭代顺序副作用，让加密与解密在分块边界消费不同数量的输入字节。
- 识别信号：自制流密码实现、跨块复用生成器，以及加密/解密两端把有限迭代器放在 `zip` 不同位置。
- 复用要点：认证标签只能保证密文未被外部篡改，不能修复实现自身的不对称；审计流式代码时要跟踪迭代器在终止判断中是否被多消费。
