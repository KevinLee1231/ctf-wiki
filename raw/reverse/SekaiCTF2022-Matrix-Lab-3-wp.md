# Matrix Lab 3

## 题目简述

题目给出一套带 VectorBlox 模拟库的 C 工程。程序读取 64 字节 flag，先在 8 位向量上执行异或、条件乘法与 $8\times8$ 转置，再把每个 8 字节行块送入 44 轮 Simon64/128，最后与 64 字节常量比较。

README 链接的 [VectorBlox 教程](https://github.com/riscvarchive/riscv-4th-workshop-tutorials/tree/master/vectorblox_tutorial) 说明了 MXP scratchpad、向量长度及二维/三维步长模型；[Simon 参考实现](https://github.com/openluopworld/block-ciphers/tree/master/simon) 给出了轻量级分组密码的基本结构。不过题目已经附带实际使用的 `vbxapi` 和 `utils.c`，解题时必须以本地实现的步长、字节序、轮数和密钥扩展为准。

## 解题过程

`official.c` 对输入格式的限制为：

```c
strlen(command) == 0x40
strncmp(command, "SEKAI{", 6) == 0
command[63] == '}'
```

随后对 64 个字节依次执行：

```c
vbx(VSBU, VXOR, v_A, v_A, 0x13);
vbx(SVBU, VMOV, v_B, 2, 0);
vbx(VSBU, VSLT, v_D, v_A, 97);
vbx(VVBU, VSUB, v_B, v_B, v_D);
vbx(VVBU, VMUL, v_C, v_A, v_B);
manipulate(v_O, v_C, 8);
```

令原字节为 $x$，$a=x\oplus0x13$。`VSLT` 在 $a<97$ 时产生 1，否则产生 0，所以乘数为：

$$
m=\begin{cases}
1,&a<97\\
2,&a\ge97
\end{cases}
$$

向量元素类型为 `vbx_ubyte_t`，乘法结果按 8 位截断，即 $c=a\cdot m\bmod256$。

`manipulate()` 的参数为：

```c
vbx_set_vl(1, N, N);
vbx_set_2D(N, 1, 0);
vbx_set_3D(1, N, 0);
vbx(VVBU, VMOV, v_dst, v_src, 0);
```

结合 VectorBlox 的二维、三维地址步长，可知源矩阵以行方向读取、目标矩阵以列方向落位，效果就是转置 $8\times8$ 字节矩阵。

程序声称使用随机单次密钥，但 RNG 固定初始化为 `0xdeadbeef`。按源码中的 32 位溢出规则复现：

```c
seed = seed * 110515245 + 114514;
return (seed / 65536) % 32768;
```

并过滤到 ASCII 33 至 126 后，16 字节密钥恒为：

```text
<Zp,TdSzQ95eCr}*
```

`utils.c` 将它按小端序解释为 4 个 32 位字，使用 `z3` 常量扩展出 44 个轮密钥。加密每两轮更新一次左右字：

```c
left  ^= round_key[i]     ^ f(right);
right ^= round_key[i + 1] ^ f(left);
```

因此解密只需从第 43 轮倒序执行，先撤销 `right`，再撤销 `left`。下面的求解器保留了题目实现的字节序和轮函数；为节省篇幅，`Z3` 就是 `utils.c` 中同名的 62 位数组。

```python
import struct

SECRET = bytes.fromhex(
    "1ecb87c1b47670b999addf841e622566"
    "385072e3f15f6c000cefaf94c603c4b1"
    "7f9618b37f94540ac7f8c2f119e5dabf"
    "d78fcebb0e7de8ddc2ca29cbc1230366"
)

Z3 = [
    1, 1, 0, 1, 1, 0, 1, 1, 1, 0, 1, 0, 1, 1, 0, 0,
    0, 1, 1, 0, 0, 1, 0, 1, 1, 1, 1, 0, 0, 0, 0, 0,
    0, 1, 0, 0, 1, 0, 0, 0, 1, 0, 1, 0, 0, 1, 1, 1,
    0, 0, 1, 1, 0, 1, 0, 0, 0, 0, 1, 1, 1, 1,
]

MASK = 0xffffffff

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & MASK

def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & MASK

def f(x):
    return (rol32(x, 1) & rol32(x, 8)) ^ rol32(x, 2)

def make_key():
    seed = 0xdeadbeef
    key = []
    while len(key) < 16:
        seed = (seed * 110515245 + 114514) & MASK
        value = ((seed // 65536) % 32768) % 256
        if 33 <= value <= 126:
            key.append(value)
    return bytes(key)

def expand_key(key):
    round_keys = list(struct.unpack("<4I", key))
    for i in range(4, 44):
        temp = ror32(round_keys[i - 1], 3) ^ round_keys[i - 3]
        temp ^= ror32(temp, 1)
        value = 0xfffffffc ^ round_keys[i - 4] ^ temp
        if Z3[(i - 4) % 62]:
            value ^= 1
        round_keys.append(value & MASK)
    return round_keys

ROUND_KEYS = expand_key(make_key())

def decrypt_block(block):
    left, right = struct.unpack("<2I", block)
    for i in range(43, -1, -2):
        right = (right ^ ROUND_KEYS[i] ^ f(left)) & MASK
        left = (left ^ ROUND_KEYS[i - 1] ^ f(right)) & MASK
    return struct.pack("<2I", left, right)

# 先逐块撤销 Simon。
vector_output = b"".join(
    decrypt_block(SECRET[i:i + 8])
    for i in range(0, 64, 8)
)

# 转置是自身的逆操作。
transformed = bytes(
    vector_output[row * 8 + col]
    for col in range(8)
    for row in range(8)
)

# 条件乘 2 在模 256 下不宜直接求逆；枚举可打印原字节。
known = b"SEKAI{"
flag = bytearray()
for index, target in enumerate(transformed):
    candidates = []
    for ch in range(32, 127):
        value = ch ^ 0x13
        factor = 1 if value < 97 else 2
        if value * factor & 0xff == target:
            candidates.append(ch)

    if index < len(known):
        candidates = [ch for ch in candidates if ch == known[index]]
    if index == 63:
        candidates = [ch for ch in candidates if ch == ord("}")]

    assert len(candidates) == 1
    flag.append(candidates[0])

print(flag.decode())
```

运行后得到：

```text
SEKAI{y4y_u_p4ss3d_ScR4TcHp4D_t35t_w1th_V3ct0rB10x_4nd_51M0N_xD}
```

## 方法总结

题目的混淆感来自 VectorBlox API，但核心仍是字节矩阵变换。确认 `vbx_set_2D`、`vbx_set_3D` 的源/目标步长后，复杂的向量操作可以压缩成一句“转置 $8\times8$ 矩阵”。

后半段应严格照本地 `utils.c` 复现 Simon：本题使用小端 32 位字、Simon64/128 的 44 轮配置和固定 RNG 密钥。最后的条件乘法在模 256 下不总是单射，因此枚举可打印字节比贸然计算模逆更稳妥；flag 前缀、结尾和正向复算可以进一步消除歧义。
