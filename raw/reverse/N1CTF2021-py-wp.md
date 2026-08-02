# N1CTF 2021 - py

## 题目简述

附件是 PyInstaller 打包的 Python 3.5 程序。入口模块要求输入 28 个十六进制字符，并导入 `L` 与 `var`；真正的校验逻辑藏在经过自定义 opcode、异或加密和运行时自修改的 `L.pyc` 中。恢复代码后可知，输入被解释为椭圆曲线离散对数，曲线阶平滑，可用 Pohlig-Hellman 求解。

## 解题过程

### 解包 PyInstaller 与 PYZ

先用 `pyinstxtractor` 解开外层可执行文件，再用 `uncompyle6` 反编译入口 `0a5n.pyc`。入口只暴露了三条重要信息：

1. flag 主体是 28 位十六进制字符串。
2. 程序导入 `L` 和 `var` 模块。
3. `PYZ-00.pyz` 仍需按 PyInstaller 的归档格式解压。

可复用 PyInstaller 自带的 `ZlibArchiveReader`，按 TOC 定位每个对象、解密并 zlib 解压，再补上 Python 3.5 的 pyc 头。`var.pyc` 可以正常反编译，得到曲线参数和目标点；原 WP 的第一张终端截图只是这些整数的输出，因此无需保留图片。

### 还原被修改和加密的字节码

检查 `opcode.pyc` 可发现运算符编号被交换，例如反编译器看到的 `^`、`+`、`%` 并不一定是原语义。修改 `pycdc/pycdas` 的 opcode 映射后仍无法直接反编译 `L.pyc`，但反汇编中可以看到两段大整数数组经固定异或后传给 `exec`：

```python
stage1 = "".join(chr(v ^ 2) for v in z1)
stage2 = "".join(chr(v ^ 4) for v in z2)
```

第一段展开后调用 `ptrace`，在未被调试时得到初始密钥 1；第二段直接修改若干函数 `co_code` 所在内存。六个目标函数依次使用密钥 1 到 6 异或解密：模逆、最高位、椭圆曲线点加、倍点、标量乘和输入转换。

无需真的执行危险的内存自修改。用 `marshal` 读出 `L.pyc` 的顶层 code object，递归定位相应的子 code object，然后离线异或：

```python
def decrypt(bytecode, key):
    return bytes(v ^ key for v in bytecode)

# 按 co_consts 中的真实位置分别替换六个函数的 co_code，key 为 1..6
```

写回新的 pyc 后，再把不可见的 `co_varnames` 机械改成 `var_0`、`var_1` 等可打印名称，即可让反编译器恢复主体源码。原 WP 剩余三张图片分别是变量常量、字节码反汇编和反编译代码截图，信息都已由上述文本覆盖。

### 还原椭圆曲线问题

恢复后的构造函数会再次变换 `var` 中的常量，最终曲线为

$$
E: y^2=x^3+ax+b\pmod p,
$$

其中：

```python
p  = 1461501637330902918203684832716283019651637554291
a  = 1461501637330902918203684832716283019651637554289
b  = 33
Gx = 1409958218732090440323571427282941405264992526638
Gy = 1003170987214086410878112234291438209997203387689
Qx = 418314664634765473100948993230460851448740309937
Qy = 1014751162621960915383962534690487909615594365554
```

输入转换函数按十六进制半字节反转 flag 主体，再解释成整数 $k$；校验条件是 $Q=kG$。

### Pohlig-Hellman 求离散对数

曲线阶可分解成若干较小素数幂。对每个 $r_i^{e_i}\mid n$，投影到对应子群并求局部离散对数：

```python
E = EllipticCurve(GF(p), [a, b])
G = E(Gx, Gy)
Q = E(Qx, Qy)
n = E.order()

mods, rems = [], []
for prime, exponent in factor(n):
    modulus = prime^exponent
    Gi = (n // modulus) * G
    Qi = (n // modulus) * Q
    rems.append(discrete_log(Qi, Gi, operation='+'))
    mods.append(modulus)

k = crt(rems, mods)
body = hex(k)[2:][::-1]
print(f"n1ctf{{{body}}}")
```

最终得到：

```text
n1ctf{304e6e4f3155756f493169304c6c}
```

## 方法总结

这道题的难点是多层 Python 保护，而不是单纯跑一个离散对数。可靠的顺序是：先解 PyInstaller/PYZ，依据实际 `opcode.pyc` 修正语义，再静态解开运行时异或的 `co_code`，最后才把业务逻辑化为数学模型。碰到平滑曲线阶时，用 Pohlig-Hellman 分解子群问题并由 CRT 合并即可。
