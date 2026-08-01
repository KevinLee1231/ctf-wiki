# GlacierCTF2022 - Wats Up

## 题目简述

附件是一个很小的 WebAssembly 模块，内含固定 mask、26 字节密文和一个受 magic number 保护的变换函数。目标是从 Wasm 的线性内存与循环逻辑还原输入数据，再模拟算法得到 secret key，也就是 flag。

## 解题过程

反编译后可见函数只在参数等于 `0xCAFEBABE` 且输入长度正确时运行。第 $i$ 个字节的核心变换为：

$$
x_i=\left(\frac{(i\ll3)\mathbin{\&}255}{4}\right)\oplus c_i\oplus m_i,
$$

随后把结果 13 加 37、结果 0 加 50。线性内存中的 mask 是重复的 ASCII `42`，密文十六进制为：

```text
535c5157555d4a5f50465b73187b657346554f254f475e494a7d
```

不必重新编译 Wasm，逐字节模拟即可：

```python
ciphertext = bytes.fromhex(
    "535c5157555d4a5f50465b73187b657346554f254f475e494a7d"
)
mask = b"42" * len(ciphertext)
plain = bytearray()

for i, value in enumerate(ciphertext):
    x = (((i << 3) & 0xff) >> 2) ^ value ^ mask[i]
    if x == 13:
        x += 37
    if x == 0:
        x += 50
    plain.append(x)

print(plain.decode())
```

输出为：

```text
glacierctf{W4SM_RE_1S_FUN}
```

## 方法总结

Wasm 只是载体，分析顺序仍是定位数据段、确认导出函数与参数门槛、翻译循环和验证长度。这里没有真正的密码学原语，只有位置相关 XOR 与两个字节修正规则；逐指令动态调试并非必要，短小的等价模拟器更容易复核。
