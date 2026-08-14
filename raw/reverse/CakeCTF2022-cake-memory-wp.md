# CakeCTF 2022 Cake Memory Writeup

## 题目简述

题目提供一个 Rust 编写的视听记忆游戏。玩家需要按照展示顺序点击颜色、文字和声音对应的按钮，后期序列长度达到 100，且程序还维护了额外计数器检测简单内存篡改。

公开仓库没有官方 solver，但保留了完整源码。与其实际完成所有记忆轮次，更稳定的做法是逆向最终 `drop_flag` 逻辑，直接复现嵌入式密文的解密过程。

## 解题过程

### 确定最终轮次状态

每次大轮次开始时，程序执行：

```rust
self.round_num = self.round_num * 2 + 1;
```

初始值为 0，所以轮次依次为：

```text
1 -> 3 -> 7 -> 15 -> 31
```

当 `round_num == 31` 且最后一轮完成时，界面调用 `drop_flag()` 显示 flag。因此离线解密需要代入的轮次字节就是 `31`。

### 还原 drop_flag

`drop_flag` 中包含 35 字节密文和字符串密钥 `bunzo`。代码先执行标准 RC4 KSA，再执行 PRGA；每个输出字节还额外与 `round_num` 异或：

$$
P_i=C_i\oplus K_i\oplus31.
$$

等价 Python 实现如下：

```python
cipher = [
    120, 239, 96, 134, 201, 74, 183, 101, 178, 169,
    227, 171, 1, 250, 125, 131, 130, 49, 146, 22,
    131, 117, 153, 49, 106, 130, 65, 20, 77, 143,
    190, 198, 134, 28, 68,
]
key = b"bunzo"

# RC4 KSA
state = list(range(256))
j = 0
for i in range(256):
    j = (j + state[i] + key[i % len(key)]) & 0xff
    state[i], state[j] = state[j], state[i]

# RC4 PRGA，并复现源码中的 round_num 异或。
out = bytearray()
i = j = 0
for value in cipher:
    i = (i + 1) & 0xff
    j = (j + state[i]) & 0xff
    state[i], state[j] = state[j], state[i]
    stream = state[(state[i] + state[j]) & 0xff]
    out.append(value ^ stream ^ 31)

print(out.decode())
```

输出为：

```text
CakeCTF{Do_you_have_Chromesthesia?}
```

源码中还残留了右键增加 `mem_count` 的调试分支，但程序同时检查 `mem_count_ == mem_count * 77 + 1`。盲目连续右键会在下一帧触发反作弊；它并不比直接恢复 `drop_flag` 更可靠。

## 方法总结

这题的附件包含大量音频，但 flag 并不隐藏在音频频谱或元数据中，决定性障碍是恢复程序状态和解密函数。因此应归为 Reverse，而不是 Stego。

分析交互式游戏时，应先定位胜利分支和最终输出函数，再判断是否真的需要重放完整游戏。这里轮次状态是确定的，密钥与密文也都在程序内，复现 30 余行解密逻辑即可形成完整证据闭环。
