# filter_flag

## 题目简述

服务每个会话以随机 AES key 与 IV 加密一个恰好 $80$ 字节、即五个 $16$ 字节块的 flag，然后允许提交同长度 ciphertext 请求解密。唯一禁止条件是 ciphertext 与最初给出的 ciphertext 完全相同。解密结果按块过滤：整块均为可打印 ASCII 才显示，否则显示十六个问号。flag 内容包含随会话变化的部分，因此题面只给出正则格式，而不是固定的一条明文。

实现名义上是 PCBC，实际链状态为 $S_i=P_i\oplus C_i$：

$$
C_i=E_K(P_i\oplus S_{i-1}),\qquad
P_i=D_K(C_i)\oplus S_{i-1},\qquad
S_i=P_i\oplus C_i.
$$

这里状态对前序 ciphertext/plaintext 的 XOR 是可交换的。只要让某一位置之前的块集合不变，即使它们的顺序被交换，状态也会恢复。攻击的核心是这个 PCBC 状态可交换性与过宽的“只禁原密文”过滤，故归入 `crypto`。

## 解题过程

### 用替换尾块恢复前四块

收到原密文后拆为 $C_1,C_2,C_3,C_4,C_5$。提交：

$$
C_1\parallel C_2\parallel C_3\parallel C_4\parallel C_1.
$$

它和原密文不同，因而绕过精确匹配过滤；前四个 ciphertext 和其前序状态又完全没变，所以返回文本的前 $64$ 字符正是 $P_1\parallel P_2\parallel P_3\parallel P_4$。末块使用错误的 $C_1$，通常会被打印过滤掩成问号，但无需依赖该块。

### 交换前两块，使状态在第二块后恢复

第二个查询为：

$$
C_2\parallel C_1\parallel C_3\parallel C_4\parallel C_5.
$$

记 $D_i=D_K(C_i)$。原始状态为：

$$
S_2=D_2\oplus D_1\oplus IV\oplus C_1\oplus C_2.
$$

交换后，前两块的 plaintext 会损坏，但第二块后的状态是：

$$
S'_2=D_1\oplus D_2\oplus IV\oplus C_2\oplus C_1=S_2.
$$

因此从 $C_3$ 开始，所有后续状态都重新与原始解密一致，返回的第 $3$、$4$、$5$ 块都是原文。只需取这次响应最后 $16$ 个字符，就得到 $P_5$。相比于删除所有 `?`，按固定块位置切片更稳健，因为偶然可打印的错误块不会改变偏移。

### 同一会话内拼接并提交

攻击必须在获取密文的同一连接内完成，因为 key、IV 和时间相关 flag 都是会话态。核心交互如下，`io` 是连接到题目服务的 tube：

```python
enc = bytes.fromhex(io.recvuntil(b"\n").split(b": ", 1)[1].strip().decode())
c1, c2, c3, c4, c5 = [enc[i:i + 16] for i in range(0, 80, 16)]

io.sendlineafter(b"decrypt: ", (c1 + c2 + c3 + c4 + c1).hex().encode())
first = io.recvuntil(b"\n").split(b"Decrypted: ", 1)[1].rstrip(b"\n").decode()

io.sendlineafter(b"decrypt: ", (c2 + c1 + c3 + c4 + c5).hex().encode())
second = io.recvuntil(b"\n").split(b"Decrypted: ", 1)[1].rstrip(b"\n").decode()

flag = first[:64] + second[-16:]
print(flag)
```

输出是当前会话的完整 flag，并符合题面格式 `grey\{7c0c6a199cda0a96b55e74c5e1.{32}_W4h_uR_Q_gUd_4h\}`。这也验证了五个块均由合法解密产生，而不是猜测动态 tag。

## 方法总结

- 核心技巧：利用 PCBC 状态 $P_i\oplus C_i$ 的 XOR 可交换性，在保持后续状态不变的同时构造不同 ciphertext。
- 识别信号：解密 oracle 仅禁止“原密文完全相等”、仍允许等长重排/替换，且按块泄露可打印明文时，应先推导模式的状态恢复点而不是把它当作普通 CBC padding oracle。
- 复用要点：先用替换无关尾块保留一个原文前缀，再交换早期块恢复链状态以读取后缀。多次查询的密钥如果是会话随机的，必须在同一连接中完成并按固定长度切片拼接。
