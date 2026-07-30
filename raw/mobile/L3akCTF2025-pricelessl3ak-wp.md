# L3akCTF 2025 PricelessL3ak Writeup

## 题目简述

PricelessL3ak 表面上是一个直接计算 SHA-256 的 flag 检查器，但 APK 中还存在一个导出的透明 Activity。利用同一 Activity 的 `onCreate()` 与 `onNewIntent()` 生命周期差异，可以让它先读取加密的 VM 字节码，再把第二次 Intent 的 flags 当作密钥解密。解出的指令通过 Android `Parcel` 和 `HandlerThread` 送入自定义虚拟机，最终对 30 字节输入执行两轮变换并比较目标数组。

本题的决定性入口是 Android 导出组件、Intent flags 和 Activity 生命周期，所以归入 Mobile；后半段的 VM 逆向是利用该入口后必须完成的数据恢复阶段。

## 解题过程

### 识别导出 Activity

检查 `AndroidManifest.xml`，除正常的主界面外还能看到：

```xml
<activity
    android:theme="@android:style/Theme.Translucent.NoTitleBar"
    android:name="ctf.l3akctf.pricelessl3ak.h1832fla12"
    android:exported="true" />
```

`h1832fla12.onCreate()` 只接受 action `BINGO`。条件满足时，它读取 `assets/data.enc` 并将字节保存在 Activity 对象中：

```java
if (!"BINGO".equals(getIntent().getAction())) {
    finish();
    return;
}
InputStream in = getAssets().open("data.enc");
byte[] data = new byte[in.available()];
in.read(data);
```

`onNewIntent()` 则只处理 action `BANGO`，并要求前一步的数据、非零 Intent flags 和字符串 extra `f` 都存在：

```java
if ("BANGO".equals(intent.getAction())
        && encryptedData != null
        && intent.getFlags() != 0
        && intent.getStringExtra("f") != null) {
    int seed = intent.getFlags();
    // 使用 seed 解密 data.enc，再用 extra "f" 作为待检查 flag
}
```

关键在于必须复用原 Activity 实例。第一次启动加载数据，第二次通过 `FLAG_ACTIVITY_SINGLE_TOP` 让系统调用已有实例的 `onNewIntent()`，而不是新建一个对象。

### 构造两阶段 Intent

可编写一个最小 Android 应用，按钮点击后执行：

```java
ComponentName target = new ComponentName(
    "ctf.l3akctf.pricelessl3ak",
    "ctf.l3akctf.pricelessl3ak.h1832fla12"
);

Intent first = new Intent();
first.setComponent(target);
first.setAction("BINGO");
first.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(first);

Intent second = new Intent();
second.setComponent(target);
second.setAction("BANGO");
second.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
second.addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP);
second.putExtra("f", "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa");

new Handler().postDelayed(() -> startActivity(second), 1000);
```

第二次 Intent 的 flags 为：

```text
FLAG_ACTIVITY_NEW_TASK  = 0x10000000
FLAG_ACTIVITY_SINGLE_TOP = 0x20000000
seed = 0x10000000 | 0x20000000 = 0x30000000
```

随意输入会弹出失败提示，但这已经证明生命周期链路和后台检查器被成功触发。

### 解密并解析 VM 字节码

`data.enc` 的解密过程依次为：

1. 从尾到头执行 `data[i] ^= data[i - 1]`；
2. 第 $i$ 字节循环右移 $(i \bmod 7)+1$ 位；
3. 减去 $(i \times 0x13 + (seed \mathbin{\&} 0xff)) \bmod 256$；
4. 异或 `seed` 中由 $i \bmod 4$ 选出的一个字节。

对应的恢复代码为：

```python
import struct

SEED = 0x30000000

def ror8(value, amount):
    return ((value >> amount) | (value << (8 - amount))) & 0xff

data = bytearray(open("data.enc", "rb").read())

for i in range(len(data) - 1, 0, -1):
    data[i] ^= data[i - 1]

for i in range(len(data)):
    data[i] = ror8(data[i], (i % 7) + 1)

for i in range(len(data)):
    addend = (i * 0x13 + (SEED & 0xff)) & 0xff
    data[i] = (data[i] - addend) & 0xff

for i in range(len(data)):
    key_byte = (SEED >> ((i % 4) * 8)) & 0xff
    data[i] ^= key_byte

instructions = [
    struct.unpack_from("<BBBI", data, offset)
    for offset in range(0, len(data), 7)
]
print(len(instructions), instructions[:5])
```

实际 APK 中 `data.enc` 长 15169 字节，恰好解析为 2167 条指令，没有剩余字节。每条指令都是 7 字节小端结构：

```text
opcode:1 | reg1:1 | reg2:1 | immediate:4
```

前五条解析结果为：

```text
(0x32, 0, 0, 0)       LOAD_CHAR
(0x28, 0, 0, 32)      CMP
(0x43, 0, 0, 7)       JL
(0x28, 0, 0, 126)     CMP
(0x44, 0, 0, 7)       JG
```

这也从实际字节码侧验证了解密 seed 与指令格式。

### 理清 Parcel 与 VM 调度

解密后的指令对象实现了 `Parcelable`。程序把候选 flag 与指令列表装入消息对象，经 `Parcel` 发送给名为 `BackgroundProcessor` 的 `HandlerThread`：

```text
消息类型 0x1337：执行 VM
结果类型 0x1338：返回布尔结果
```

这不是跨应用的远程服务，而是应用内部借助 Parcel 序列化和 Handler 消息队列隐藏调用关系。最终执行器维护 16 个寄存器、内存映射、程序计数器和比较标志，并解释 `LOAD`、`STORE`、`LOAD_CHAR`、`XOR`、`MUL`、`ROL`、`ROR`、条件跳转等 opcode。

### 化简三段校验逻辑

VM 首先要求输入长度为 30。对每个字符 $c_i$，若它位于可打印 ASCII 区间 $[0x20,0x7e]$，则：

$$
n_i = c_i - 0x20
$$

否则将 $n_i$ 置为 `0x1f`。各字节保存到 `0x1000+i`。

第二阶段计算：

$$
r_i = \operatorname{ROL8}\left(
n_i \oplus ((i \times 0x17) \mathbin{\&} 0xff)
\oplus (i^2 \mathbin{\&} 0xff),
(i \bmod 8)+1
\right)
$$

结果保存到 `0x1100+i`。

第三阶段引入跨位置依赖。令：

$$
d_{1,i}=r_{(3i)\bmod 30},\qquad
d_{2,i}=r_{(7i)\bmod 30}
$$

则：

$$
v_i =
\left((r_i+d_{1,i}^2)\bmod 256\right)
\oplus
\left((d_{2,i}\times r_i)\bmod 256\right)
$$

$$
o_i=\operatorname{ROR8}\left(v_i,(d_{1,i}+d_{2,i})\bmod 8\right)
$$

VM 将 $o_i$ 与下面的 30 字节目标逐一比较：

```python
target = [
    0xd8, 0x50, 0x23, 0x16, 0x81, 0xb0, 0xe7, 0x4c,
    0x9a, 0xb5, 0x4b, 0xd9, 0x2a, 0x98, 0x58, 0x14,
    0xea, 0xc6, 0x90, 0x51, 0x20, 0x48, 0x43, 0x72,
    0x28, 0x85, 0x4b, 0xad, 0xa5, 0x80,
]
```

任一位置不匹配都会把结果寄存器 `r14` 清零。

### 用 Z3 求候选并以 SHA-256 消歧

跨位置依赖使逐字节逆推不再独立，适合直接用 8 位 BitVec 建模。下面是核心求解器：

```python
import hashlib
from z3 import (
    BitVec, BitVecVal, If, Or, RotateLeft, Solver, sat
)

target = [
    0xd8, 0x50, 0x23, 0x16, 0x81, 0xb0, 0xe7, 0x4c,
    0x9a, 0xb5, 0x4b, 0xd9, 0x2a, 0x98, 0x58, 0x14,
    0xea, 0xc6, 0x90, 0x51, 0x20, 0x48, 0x43, 0x72,
    0x28, 0x85, 0x4b, 0xad, 0xa5, 0x80,
]
expected_hash = (
    "f3bdd9f68a198756b96c5cf8207db63"
    "a11507e50fb0d29be609ff678ef721935"
)
n = len(target)
s = Solver()

chars = [BitVec(f"char_{i}", 8) for i in range(n)]
norm = [BitVec(f"norm_{i}", 8) for i in range(n)]
r1 = [BitVec(f"r1_{i}", 8) for i in range(n)]

def rol8_dynamic(value, amount):
    result = RotateLeft(value, 0)
    for rotation in range(1, 8):
        result = If(
            amount == BitVecVal(rotation, 8),
            RotateLeft(value, rotation),
            result,
        )
    return result

for i in range(n):
    s.add(chars[i] >= 0x20, chars[i] <= 0x7e)
    s.add(norm[i] == chars[i] - 0x20)
    mask = ((i * 0x17) & 0xff) ^ ((i * i) & 0xff)
    s.add(
        r1[i]
        == RotateLeft(norm[i] ^ BitVecVal(mask, 8), (i % 8) + 1)
    )

for i in range(n):
    a = r1[i]
    dep1 = r1[(i * 3) % n]
    dep2 = r1[(i * 7) % n]
    rotation = (dep1 + dep2) & BitVecVal(7, 8)
    polynomial = (a + dep1 * dep1) ^ (dep2 * a)

    # target[i] = ROR8(polynomial, rotation)
    s.add(
        rol8_dynamic(BitVecVal(target[i], 8), rotation)
        == polynomial
    )

known = b"L3AK{"
for i, value in enumerate(known):
    s.add(chars[i] == value)
s.add(chars[-1] == ord("}"))

while s.check() == sat:
    model = s.model()
    candidate = bytes(model.eval(char).as_long() for char in chars)

    if hashlib.sha256(candidate).hexdigest() == expected_hash:
        print(candidate.decode())
        break

    s.add(Or([
        chars[i] != BitVecVal(candidate[i], 8)
        for i in range(n)
    ]))
```

VM 约束并不保证唯一字符串，实际会出现约 3000 个候选。主界面中硬编码的 SHA-256：

```text
f3bdd9f68a198756b96c5cf8207db63a11507e50fb0d29be609ff678ef721935
```

正好可作为第二重判据。最终得到：

```text
L3AK{P4rc3l_cycl3_1Nt3nt_VM!!}
```

该字符串长度为 30，重新计算 SHA-256 后与应用常量完全一致。

## 方法总结

本题的完整链条是“导出 Activity → 两阶段生命周期复用 → Intent flags 派生 seed → 解密 VM 字节码 → Parcel/Handler 调度 → 约束求解 → SHA-256 消歧”。单看主界面的哈希比较无法直接逆出输入，单看 `data.enc` 也不知道正确 seed；必须把 Android 组件行为和后续逆向结合起来。

分析导出 Activity 时，要同时检查 `onCreate()`、`onNewIntent()`、launch flags 和对象内保留的状态。遇到自定义 VM，则应先把字节码解析为稳定结构，再按内存区域和重复指令模式归纳高层公式。最后，本题提醒我们：约束系统存在解并不等于答案唯一；程序中另一个看似多余的哈希常量，可能正是作者留下的唯一性判据。
