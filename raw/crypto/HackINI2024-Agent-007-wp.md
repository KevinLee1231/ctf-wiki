# HackINI2024 Agent 007

## 题目简述

题目给出一条 Cisco IOS 配置：

```text
enable password 7 0832444B05150816060E1F1F232437172466231838130D024B
```

其中 `7` 表示密码采用 Cisco Type 7 混淆。Type 7 不是单向哈希，而是使用固定密钥表进行异或，因此只要还原其编码流程就能恢复明文。本题附件的密文末尾存在损坏，前半段仍可验证，最终 flag 以官方解答给出的预期明文为准。

## 解题过程

Type 7 字符串的前两位十进制数字是密钥表起始偏移。本题前缀为 `08`，所以从固定密钥表 `dsfd;kfoA,.iyewrkldJKDHSUB` 的第 8 个位置开始，依次将后续每个十六进制字节与密钥表字符异或：

```python
KEY = b"dsfd;kfoA,.iyewrkldJKDHSUB"
ciphertext = "0832444B05150816060E1F1F232437172466231838130D024B"

seed = int(ciphertext[:2])
plain = bytes(
    int(ciphertext[i:i + 2], 16)
    ^ KEY[(seed + (i - 2) // 2) % len(KEY)]
    for i in range(2, len(ciphertext), 2)
)
print(repr(plain))
```

实际得到：

```text
b'shellmates{ios_w3a|Kui9 '
```

结果的 `shellmates{ios_w3a` 前缀合理，但末尾既没有闭合花括号，也与官方答案不符，说明题目中给出的 Type 7 密文后缀已经损坏。这不是解码算法的问题：用相同的 `08` 偏移重新编码官方预期明文 `shellmates{ios_w3ak_enc}`，应得到：

```text
0832444B05150816060E1F1F232437172466230F2C030A5816
```

它与题目密文在后段发生分歧。结合官方解答，最终 flag 为：

```text
shellmates{ios_w3ak_enc}
```

## 方法总结

Cisco Type 7 只是可逆的异或混淆，不能用于安全保存口令。解题时应先识别格式中的种子，再按固定密钥表逐字节异或。遇到解码结果只在尾部异常时，还应通过反向编码校验输入是否损坏；本题只能从附件稳定恢复 flag 前缀，完整答案依赖官方解答所给的预期明文，不能把损坏后的乱码伪装成可复现结果。
