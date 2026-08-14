# nande

## 题目简述

这题是位级电路语义恢复：将输入字符串按 bit 流送入 `CIRCUIT()`，经过固定次数变换后与 `AnswerSequence[0x100]` 对比。

`main.c` 中关键点：

- `flag` 长度固定 `0x20`。
- 每字节按位拆为 8bit 小端位序存到 `InputSequence`。
- `CIRCUIT` 重复 0x1234 轮，每轮对前 255 项执行一次 `MODULE`，最后一项与常数 1 处理。
- `MODULE` 是 NAND 逻辑复合，输出为：

```c
void MODULE(bit a, bit b, bit*x) {
    bit t1, t2, t3;
    NAND(a, b, &t1);
    NAND(a, t1, &t2);
    NAND(b, t1, &t3);
    NAND(t2, t3, x);
}
```

这四次 NAND 恰好实现 $x=a\oplus b$，所以一轮电路的精确关系是：

$$
\begin{aligned}
out_i &= in_i\oplus in_{i+1} && (0\le i<255),\\
out_{255} &= in_{255}\oplus 1.
\end{aligned}
$$

整个变换是可逆的仿射变换，并不存在需要穷举的非线性秘密。题目给出了专用 `solution/solve.py`，解题重点是从后向前恢复上一轮状态。

## 解题过程

### 关键观察

由最后一个关系先得到 $in_{255}=out_{255}\oplus1$，再按 $i=254,253,\ldots,0$ 使用 $in_i=out_i\oplus in_{i+1}$。官方 `solution/solve.py` 从 `distfiles/nand.exe` 的文件偏移 `0x1c600` 读出长度 `0x100` 的 `AnswerSequence`，并原地执行

```python
seq[0xff] ^= 1
for i in range(0xfe, -1, -1):
    seq[i] ^= seq[i+1]
```

重复 `0x1234` 次后按位组装即可得到原始 flag。

### 求解步骤

1. 按偏移 `0x1c600` 读取 0x100 字节。
2. 循环 `0x1234` 次执行：
   - `seq[0xff] ^= 1`
   - 从后向前 `seq[i] ^= seq[i+1]`
3. 每 8 位转一次字节，按小端位序拼接成 0x20 字符。

复现脚本如下：

```python
with open("../distfiles/nand.exe", "rb") as f:
    f.seek(0x0001c600)
    seq = list(f.read(0x100))

for _ in range(0x1234):
    seq[0xff] ^= 1
    for i in range(0xfe, -1, -1):
        seq[i] ^= seq[i + 1]

flag = ""
for i in range(0, 0x100, 8):
    c = 0
    for j in range(8):
        c |= seq[i + j] << j
    flag += chr(c)
print(flag)
```

### 验证

按上面脚本还原得到：

```text
CakeCTF{h2fsCHAo3xOsBZefcWudTa4}
```

该结果与 `task.yml` 中的官方 flag 一致。

## 方法总结

- 核心技巧：先把四门 NAND 化简成 XOR，再推导单轮状态变换的逆式，连续逆推 `0x1234` 轮。
- 识别信号：有明确 `0x1234` 轮、固定位数组和解码脚本时，优先使用官方逆变换而非重建整段硬件逻辑。
- 复用要点：位序/组包方向（小端 bit）是容易出错的点；这是这类题最重要的复用注意项。
