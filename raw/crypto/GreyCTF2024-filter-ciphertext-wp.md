# filter ciphertext

## 题目简述

服务端公开 80 字节随机秘密的自定义 PCBC 密文，并提供解密接口。它拒绝原密文，还试图从用户输入中删除所有属于秘密密文的分组；只有解密结果恰好等于原秘密时才返回 flag。漏洞不在 AES，而在遍历列表时原地删除元素。

## 解题过程

解密前的过滤逻辑是：

```python
blocks = [ct[i:i + 16] for i in range(0, len(ct), 16)]
for block in blocks:
    if block in secret_enc:
        blocks.remove(block)
```

Python 的 `for` 循环按索引推进；删除当前元素后，后一个元素左移到当前索引，但下一轮会继续访问更大的索引，于是这个相邻元素被跳过。

设公开的五个密文块为 $C_0,\ldots,C_4$。把每块连续提交两次：

```python
payload = b"".join(
    secret_enc[i:i + 16] * 2
    for i in range(0, len(secret_enc), 16)
)
```

过滤每一对 $C_i,C_i$ 时，第一个副本被删除，第二个副本因索引移动而没有接受检查。五对处理完成后，列表中恰好还剩原顺序的 $C_0,\ldots,C_4$。因此后续 PCBC 解密收到的内容与 `secret_enc` 完全相同，输出自然等于 `secret`；与此同时，原始提交是十个分组，并不等于被显式拒绝的五分组密文。

服务端通过比较并返回：

```text
grey{00ps_n3v3r_m0d1fy_wh1l3_1t3r4t1ng}
```

## 方法总结

这是典型的“遍历时修改容器”漏洞。安全过滤应生成新列表，或用一次明确的条件筛选，而不能一边迭代一边 `remove`。审计密码服务时，不应只关注算法强度；输入规范化、黑名单处理和比较发生的先后顺序，同样可能让攻击者把被禁止的数据原样送入解密核心。
