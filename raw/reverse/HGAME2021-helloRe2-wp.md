# helloRe2

## 题目简述

程序要求依次输入两个 16 字节 password，最后按 `hgame{password1_password2}` 组合 flag。第一部分原本设计为 $4\times4$ 矩阵乘法校验，但编译器把常量矩阵运算折叠成了直接比较；第二部分把 password1 逐字节异或下标后作为 AES-128-CBC 密钥，在子进程中校验 password2。

## 解题过程

第一部分的源码把 16 字节输入按 $4\times4$ 矩阵解释，右乘对角矩阵：

$$
B=\operatorname{diag}(1,2,3,4)
$$

再与固定答案矩阵比较。实际发布的优化后二进制直接把输入同 128 位常量比较。IDA 中该常量按小端显示为：

```text
39383162303261336136653563306232
```

按内存字节序恢复字符串：

```python
value = 0x39383162303261336136653563306232
password1 = value.to_bytes(16, "little").decode()
print(password1)
```

得到：

```text
2b0c5e6a3a20b189
```

程序将第 $i$ 个字节与 $i$ 异或后写入共享内存，子进程把结果作为 AES 密钥：

```python
key = bytes(value ^ index for index, value in enumerate(password1.encode()))
print(key)
```

密钥为：

```text
2c2`1`0f;h8;n<66
```

IV 是从 `0x00` 到 `0x0f` 的 16 个连续字节，比较密文为 `b7fefed9077679653f4e5f62d502f67e`。该块长度正好是 16 字节，使用 CBC 且不填充：

```python
from Crypto.Cipher import AES

key = b"2c2`1`0f;h8;n<66"
iv = bytes(range(16))
ciphertext = bytes.fromhex("b7fefed9077679653f4e5f62d502f67e")
password2 = AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext).decode()
print(password2)
```

输出 `7a4ad6c5671fb313`，所以完整 flag 为：

```text
hgame{2b0c5e6a3a20b189_7a4ad6c5671fb313}
```

官方 PDF 的参数截图已全部转写为文本；完整 flag 拼接格式还可在 [PoilZero 的同期 helloRe2 复盘](https://poilzero.cn/index.php/archives/80/) 中交叉核对，正文无需依赖该链接。

## 方法总结

逆向时必须以最终二进制为准：源码中的矩阵算法可能被常量折叠，实际校验已经退化为内存常量比较。跨进程题还要追踪共享内存中的数据变换；本题的 AES key 不是 password1 原文，而是逐字节异或下标后的 16 字节结果，IV、模式和是否填充也都应从 API 参数确认。
