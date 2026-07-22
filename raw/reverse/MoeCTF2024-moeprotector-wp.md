# moeprotector

## 题目简述

这是一个 Windows x64 逆向题，使用 SEH 改变控制流，并加入调试器检测。真正的 flag 校验是三轮逐字节“异或后加 20”，需要在动态调试中绕过反调试、跟到最终比较，再逆序撤销变换。

## 解题过程

使用 x64dbg 配合 ScyllaHide 可通过主要调试器检测；题目还会检测 TitanHide，因此不要同时开启。SEH 异常被处理后程序会跳到下一段逻辑，单纯静态跟随正常控制流容易漏掉校验函数。

动态跟踪后可整理出等价伪代码：

```c
flag[0] ^= 0x31;
flag[0] += 0x31;
flag[0] -= 0x31;
flag[0] ^= 0x31;

for (int i = 0; i < 57; ++i) {
    flag[i] ^= 21 + i;
    flag[i] += 20;
}
for (int i = 0; i < 57; ++i) {
    flag[i] ^= 26 + i;
    flag[i] += 20;
}
for (int i = 0; i < 57; ++i) {
    flag[i] ^= 25 + i;
    flag[i] += 20;
}

if (memcmp(flag, hexData, 57) != 0)
    fail();
```

开头针对 `flag[0]` 的四条指令互相抵消，只是干扰项。对每轮而言，加密为 `y = (x ^ key) + 20 mod 256`，所以逆变换是先减 20，再异或；三轮还必须按 `25、26、21` 的相反顺序处理：

```python
def decrypt(hex_data):
    data = bytes(hex_data)
    for seed in (25, 26, 21):
        data = bytes(
            (((value - 20) & 0xFF) ^ ((seed + index) & 0xFF))
            for index, value in enumerate(data)
        )
    return data

# hex_data 从最终 memcmp 的另一侧提取。
# print(decrypt(hex_data))
```

从二进制最终比较位置提取 `hexData` 后运行逆变换，得到：

```text
moectf{w1Nd0Ws_S3H_15_A_g0oD_m37h0d_70_h4nd13_EXCEPTI0NS}
```

## 方法总结

SEH 既是 Windows 异常处理机制，也可用来隐藏真实控制流；反调试插件只能帮助程序跑起来，不能替代对异常落点和数据变换的追踪。撤销混合运算时，既要反转每轮内部顺序，也要反转轮次顺序。
