# SuanHash

## 题目简述

SuanHash 是一个 128-bit 状态、64-bit rate、64-bit capacity 的仿 sponge 哈希。每轮题目创建一个随机实例，允许提交 4 条消息；只要其中两条不同消息哈希相同即可进入下一轮，连续完成 500 轮后获得 flag。漏洞不在 AES 置换，而在 squeeze 输出几乎泄露了完整内部状态差分。

## 解题过程

### 写出吸收与输出公式

把某一块吸收前的状态记为

```text
S_prev = U_prev || L_prev
```

其中高、低各 64 bit。对消息块 `B = B_hi || B_lo`，代码执行：

```text
mixed = S_prev xor B
S_new = AES_k(mixed) = U_new || L_new
last_low = low64(mixed) = L_prev xor B_lo
```

最终 128-bit digest 并不是普通 sponge 的 rate 输出，而是：

```text
H_hi = U_new
H_lo = last_low xor L_new
     = L_prev xor B_lo xor L_new
```

高半直接泄露新状态的高 64 bit；低半只多混入了已知块低位和前一状态低位。

### 用两次查询恢复状态差分

令两条消息共享相同前缀，只让最后一个已填充块不同。例如：

```python
prefix = b'A' * 16
m1 = prefix + b'B'
m2 = prefix + b'C'
p1 = pad(m1)
p2 = pad(m2)
```

前缀块相同，因此处理最后一块前，两条路径的 `L_prev` 相同。查询得到 `H1`、`H2` 后：

```text
delta_hi = H1_hi xor H2_hi
delta_lo = H1_lo xor H2_lo xor p1_last_lo xor p2_last_lo
delta    = delta_hi || delta_lo
```

代入输出公式即可看到共同的 `L_prev` 被抵消，故 `delta` 正是两条消息吸收结束后的状态差：

```text
delta = S1 xor S2
```

### 追加互补块制造碰撞

随机选择一个 128-bit 块 `C`，给第二条路径追加 `C' = C xor delta`：

```text
S1 xor C
  = S2 xor delta xor C
  = S2 xor C'
```

两次 AES 置换的输入完全相同，输出状态也就完全相同。之后两条等长消息还会追加相同的 `0x80 || 0x00...` 填充块，因此最终 digest 保持相同。

单轮四次查询可以写成：

```python
MASK64 = (1 << 64) - 1

h1 = query(m1)
h2 = query(m2)

delta_hi = (h1 >> 64) ^ (h2 >> 64)
delta_lo = (h1 & MASK64) ^ (h2 & MASK64)
delta_lo ^= int.from_bytes(p1[-8:], 'big')
delta_lo ^= int.from_bytes(p2[-8:], 'big')
delta = (delta_hi << 64) | delta_lo

c = os.urandom(16)
c2 = (int.from_bytes(c, 'big') ^ delta).to_bytes(16, 'big')

m3 = p1 + c
m4 = p2 + c2
assert query(m3) == query(m4)
```

注意 `m3`、`m4` 以已经填充好的 `p1`、`p2` 开头，这是为了精确复现前两次查询结束时的状态；哈希函数随后自动附加的新填充块在两边相同。题目每轮会换一套随机 AES key 和初始配置，但攻击只使用当前轮四个输出之间的差分，所以重复上述过程 500 次即可得到 flag。

## 方法总结

安全的 sponge 不应暴露 capacity；SuanHash 却输出 `U_new` 和一个可线性消去已知量的 `L_new` 派生值，使选手能恢复完整状态差。恢复差分后，无需攻击 AES：在下一吸收块中异或同一差分，就能把两条路径送到完全相同的置换输入。审计自制哈希时应把每个输出位写成状态和消息的代数式，而不是只看内部是否用了强分组密码。
