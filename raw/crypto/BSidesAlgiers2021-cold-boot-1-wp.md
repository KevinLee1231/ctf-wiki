# Cold Boot 1

## 题目简述

题目模拟冷启动后 RAM 中部分位发生衰减的场景。附件已经从内存片段中定位出 AES-128 的 11 组轮密钥，但若干十六进制半字节以问号表示。目标是利用 AES 密钥扩展内部的冗余关系恢复第 0 轮主密钥，并提交其连续十六进制字符串的 MD5。

这一步的决定性障碍是 AES-128 key schedule 约束，而不是继续 carving 内存镜像，因此归入 crypto。

## 解题过程

把每轮 16 字节拆为四个 32 位字 $w_i$。AES-128 的扩展关系为：

$$
w_i=
\begin{cases}
w_{i-4}\oplus\operatorname{SubWord}(\operatorname{RotWord}(w_{i-1}))
\oplus\operatorname{Rcon}_{i/4},&i\bmod4=0,\\
w_{i-4}\oplus w_{i-1},&i\bmod4\ne0.
\end{cases}
$$

普通列只需要 XOR，可以从每轮最右列向左回推。例如同一字节位置满足 $w_7=w_3\oplus w_6$，所以只要三者中有两个半字节已知，就能补出第三个；附件跨越 10 轮，后续轮次又能继续消除当前轮的问号。

第 1 轮首列需要使用非线性变换。恢复第 0 轮末列为“ac 34 28 a0”后：

~~~text
RotWord(ac 34 28 a0) = 34 28 a0 ac
SubWord(...)          = 18 34 e0 91
Rcon[1] xor ...       = 19 34 e0 91
~~~

再与第 0 轮首列 XOR，就能匹配附件中的第 1 轮首列“5e ed e5 5b”。把普通列关系和这一首列变换反复应用，最终得到完整主密钥：

~~~text
47 d9 05 ca e6 e1 55 f2 6d 83 c3 a6 ac 34 28 a0
~~~

可以重新执行 AES-128 key expansion，确认生成的每个已知半字节都与附件的 11 轮数据一致。最后按题目要求对无空格的小写十六进制字符串求 MD5：

~~~python
from hashlib import md5

key_hex = "47d905cae6e155f26d83c3a6ac3428a0"
assert md5(key_hex.encode()).hexdigest() == (
    "1a9828ffb1cc9103c72fee534aec1bf1"
)
~~~

flag 为：

~~~text
shellmates{1a9828ffb1cc9103c72fee534aec1bf1}
~~~

## 方法总结

冷启动会破坏部分位，但扩展密钥往往保存了大量确定性冗余。看到多个 AES round key 同时残缺时，不应只暴力枚举 128 位主密钥；应把每个字节和半字节作为约束，使用普通列 XOR 关系双向传播，再用 RotWord、S-box 与 Rcon 检查每轮首列。最终必须重新展开全部轮密钥，而不能只因 MD5 格式正确就认为候选可信。
