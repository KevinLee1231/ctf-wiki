# SEKAI Craft

## 题目简述

附件是 Minecraft 1.21.8 世界存档。世界中有 128 个拉杆，按下按钮后，大量 datapack 命令读取拉杆状态，并用 scoreboard 在位级别执行一个修改版 XTEA。只有两组 64 位输入块加密后等于内置密文，检查才会显示成功。

题目要求把恢复出的 16 字节字符串外包 `SEKAI{}` 后提交。

## 解题过程

### 1. 从命令函数中识别位运算

每个拉杆状态被写入一个记分板变量：

```mcfunction
execute store success score leverN bit \
run execute if block <x> -59 7 minecraft:lever[powered=true]
```

随后命令用模 2 加法实现 XOR，用乘法与进位实现 32 位加法，并组合出：

$$
G(x,k)=\left(\left((x\ll4)\oplus(x\gg5)\right)+x\right)\oplus k
\pmod{2^{32}}.
$$

每轮更新为：

$$
\begin{aligned}
s&\leftarrow s+\Delta,\\
v_0&\leftarrow v_0+G(v_1,k_{s\bmod4}),\\
v_1&\leftarrow v_1+G(v_0,k_{(s\gg11)\bmod4}),
\end{aligned}
$$

共 32 轮，其中：

```text
DELTA = 0x0aef98da
```

这与标准 XTEA 相似，但常量和更新顺序必须以题目实现为准。

### 2. 提取密钥和密文

生成脚本与 datapack 中固定了四个 32 位 key：

```python
KEY_WORDS = (
    0x5f7438da,
    0xf1fa60fb,
    0x289c2239,
    0x88042cb9,
)
```

两组密文共四个 32 位字：

```python
CIPHER_WORDS = (
    0x1021d4ff,
    0xa32b2ead,
    0x04c38d5e,
    0x15a65d4b,
)
```

前 64 个拉杆形成第一块 $(v_0,v_1)$，后 64 个形成第二块；每个 32 位字均按最低位对应编号最小的拉杆。

### 3. 逆转 32 轮

初始和为：

$$
s=32\Delta\bmod2^{32}.
$$

每轮按相反顺序撤销：

```python
for _ in range(32):
    v1 = (v1 - G(v0, key[(s >> 11) & 3])) & 0xffffffff
    v0 = (v0 - G(v1, key[s & 3])) & 0xffffffff
    s = (s - DELTA) & 0xffffffff
```

分别解密两组密文，再把四个 32 位结果按小端序拼接：

```python
flag_body = b"".join(
    word.to_bytes(4, "little")
    for word in plaintext_words
)
```

得到：

```text
s3k41cr4tg00d:^)
```

最终提交：

```text
SEKAI{s3k41cr4tg00d:^)}
```

若要在世界中验证，可把解密结果的 128 个 bit 逐个转换成 `setblock ... lever[powered=true|false]` 命令。

## 方法总结

Minecraft scoreboard 虽然没有直接的 32 位无符号位运算，但可以用逐 bit XOR、半加器和选择器完整实现分组密码。逆向时应先把成千上万条命令归纳成宏：

```text
XOR、ADD32、SHL4、SHR5、SELECT_KEY
```

再识别上层轮函数。直接阅读展开后的命令流容易迷失，而从宏结构恢复修改版 XTEA 后，解密只需几十行代码。
