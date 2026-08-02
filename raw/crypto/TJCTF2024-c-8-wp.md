# c-8

## 题目简述

题目把一组 Python 与 AES 相关文件先经过自定义仿射变换，再留下一个名为 `plausibly.deniable` 的 AES-CBC 密文。自定义层按 8 字节读取明文，将其解释为整数 $x$，计算

$$
y=ax+b\pmod n,
$$

并把 $y$ 保存为 9 字节。模数是公开的素数

$$
n=18446744073709551629.
$$

加密后的 Python 源码提供了足够长的已知明文，因而可以恢复仿射参数并还原全部文件。

## 解题过程

取同一文件中两个已知 8 字节明文块 $x_1,x_2$ 及对应的 9 字节密文块 $y_1,y_2$，有

$$
a=(y_1-y_2)(x_1-x_2)^{-1}\pmod n,
$$

$$
b=y_1-ax_1\pmod n.
$$

利用 Python 文件开头稳定出现的源码片段作为 crib，官方解法恢复出：

```text
a = 1871049807465198074
b = 1776200568629335123
```

因为 $a$ 在模 $n$ 下可逆，每个分组都可直接求逆：

```python
N = 18446744073709551629
A = 1871049807465198074
B = 1776200568629335123

def decrypt_affine(data: bytes) -> bytes:
    assert len(data) % 9 == 0
    inv_a = pow(A, -1, N)
    out = bytearray()
    for offset in range(0, len(data), 9):
        y = int.from_bytes(data[offset:offset + 9], "big")
        x = ((y - B) * inv_a) % N
        out.extend(x.to_bytes(8, "big"))
    return bytes(out)
```

逐个还原附件后，解密得到的 `dec.py` 暴露了真正 AES-CBC 层使用的 key、IV 和去填充方式。用该脚本处理 `plausibly.deniable`，即可读出末尾的 flag：

```text
tjctf{sssstosssssp_sooooooooooooooooooooo_cute-13748961}
```

## 方法总结

- 仿射密码只要得到两组独立的明密文对就能恢复参数；把分组扩展为 64 bit 并不会增加安全性。
- 源代码格式是天然的已知明文，尤其是 `import`、函数定义和固定缩进，可用来校验字节序与分组边界。
- 本题是两层结构：先完整还原被仿射加密的工具文件，再依据恢复出的 key/IV 解 AES-CBC，不能把两个阶段混为一种算法。
