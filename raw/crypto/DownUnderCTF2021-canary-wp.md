# DownUnderCTF 2021 - Canary

## 题目简述

服务维护一份只初始化一次的 RC4 状态表 `S`。每轮交互先用这份状态产生 16 字节 canary，把它同时保存到全局变量和输入缓冲区的偏移 `32..47`，随后允许用户向 56 字节缓冲区写入数据。只有 canary 未被破坏，且偏移 `48..55` 等于 `247DUCTF`，程序才会输出 flag。

表面上看，填入末尾标记必然覆盖前面的未知 canary；真正的弱点在 RC4 实现，而不是传统的栈溢出。

## 解题过程

状态表中的交换函数采用 XOR-swap：

```c
void swap(unsigned char *a, unsigned char *b) {
    *a ^= *b;
    *b ^= *a;
    *a ^= *b;
}
```

当 `a == b` 时，这三次异或不会保持原值，而会把该字节清零。RC4 的 KSA 和 PRGA 都可能出现 `i == j`，而程序在每轮请求后继续复用已经被修改的 `S`。因此，每次生成 canary 都可能继续破坏状态；经过足够多轮后，前部状态退化为零，零明文对应的 16 字节密钥流也变成全零，canary 随之可预测。

官方求解脚本先发送 1250 次无效请求，让状态充分退化，然后构造完整的 56 字节输入：前 32 字节无需满足条件，接着放 16 字节零 canary，最后放 `247DUCTF`。

```python
from pwn import remote

io = remote(HOST, PORT)
io.recvuntil(b"mine?")

for _ in range(1250):
    io.sendline(b"A")
    io.recvuntil(b"you!")

payload = b"\x00" * 32
payload += b"\x00" * 16
payload += b"247DUCTF"
io.send(payload)
print(io.recvall().decode())
```

两个检查同时通过后得到：

```text
DUCTF{chirp_charp_chorp_churp}
```

## 方法总结

XOR-swap 只有在两个地址不同的前提下才与普通交换等价；别名输入会把数据清零。此题又把被 PRGA 修改的 RC4 状态跨请求复用，使单次的小概率状态破坏可以累积成确定性的全零 canary。审计自制密码实现时，除了算法公式，还应检查交换别名、状态生命周期以及同一状态是否被多个请求持续消费。
