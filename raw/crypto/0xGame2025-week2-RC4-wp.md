# RC4

## 题目简述

服务随机返回两类数据之一：flag 内部明文的 RC4 密文，或已知字符串 `b"This is keyyyyyy" * 5` 的 RC4 密文。两类输出使用同一条 RC4 密钥流，因此收集各一份即可做已知明文攻击。

RC4 key 的生成看似加入了 `os.urandom(16)`，但源码先用 AES-ECB 加密 `random_prefix || msg`，随后只取最后一个密文块。ECB 各分组独立，随机前缀只影响第一块；最后一块完全由固定 `msg` 与填充决定，所以每次进程得到的 RC4 key 实际相同。

## 解题过程

对 RC4 有 $C=P\oplus KS$。已知明文分支给出：

$$
KS=C_{known}\oplus P_{known}
$$

已知字符串比 flag 内部更长，因此恢复的密钥流足以覆盖 flag 密文。分别保存两类响应后执行：

```python
flag_ciphertext = b'...'   # “ciphertext of the flag” 分支
known_ciphertext = b'...'  # “ciphertext of the key” 分支
known_plaintext = b'This is keyyyyyy' * 5

keystream = bytes(c ^ p for c, p in zip(known_ciphertext, known_plaintext))
message = bytes(c ^ k for c, k in zip(flag_ciphertext, keystream))
print(b'0xGame{' + message + b'}')
```

若连接一次只得到一种分支，可重复启动题目进程直至收集齐两类；由于上述 ECB 分组错误，跨进程密钥流仍保持一致。

## 方法总结

- 核心技巧：利用流密码密钥流复用，通过已知明密文恢复 keystream。
- 识别信号：相同 RC4 key 从初始状态加密不同消息，其中至少一条明文已知且长度足够。
- 复用要点：分析密钥生成的真实数据依赖；随机数“出现过”不代表影响最终 key，尤其要检查 ECB 分组边界和截取位置。
