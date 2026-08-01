# Real Smooth

## 题目简述

服务用同一组 ChaCha20 密钥和 8 字节 nonce 连续加密两个已知字符串 `Slide to the left` 与 `Slide to the right`，然后重新以相同密钥和 nonce 初始化密码器，解密用户提交的密文。只有明文等于 `Criss cross, criss cross` 才返回 flag。

ChaCha20 是流密码，重复初始状态会重复密钥流；代码也没有使用 Poly1305 等消息认证。

## 解题过程

将两个已知明文和两个连续密文分别拼接。对流密码有 $C=P\oplus K$，所以可以恢复所需长度的密钥流：

$$
K=C_{known}\oplus P_{known}.
$$

再计算目标密文 $C_{target}=K\oplus P_{target}$。目标文本比已知文本短，只需截取相同长度：

```python
from pwn import xor

known_pt = b"Slide to the leftSlide to the right"
known_ct = bytes.fromhex(c1 + c2)
target = b"Criss cross, criss cross"

keystream = xor(known_ct, known_pt)
forged = xor(keystream[:len(target)], target)
sendline(forged.hex().encode())
```

服务重新初始化后使用的正是从偏移 0 开始的相同密钥流，因此伪造密文被解密为目标短语，返回：

```text
byuctf{ch4ch4_sl1d3?...n0,ch4ch4_b1tfl1p}
```

## 方法总结

- 核心技巧：利用 ChaCha20 的 key/nonce 重用，从已知明密文恢复密钥流并伪造任意等长或更短明文。
- 识别信号：流密码在相同初始状态下输出重复密钥流；已知明文加无认证解密接口通常意味着直接可塑。
- 复用要点：连续两次加密时第二段密钥流不是重新从 0 开始，必须先拼接；伪造长度不能超过已知密钥流覆盖范围。
