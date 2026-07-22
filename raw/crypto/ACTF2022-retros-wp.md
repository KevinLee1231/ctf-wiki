# retros

## 题目简述

题目要求提交 `ticket` 与 `fortune`。从程序行为和官方说明可知，二者分别作为 AES-CBC 密文与 IV：`ticket` 最长 32 字节，因此最多控制两个明文分组。解密后程序先检查 PKCS#7 填充，填充错误时输出 `Nonono, invaild ticket.`，但不会立即关闭连接；填充正确则继续把去除填充后的内容交给一台自定义 VM 执行。

最终目标是让 VM 对内部数组完成排序并执行 `check`。利用链分成两步：先把不同错误回显变成 CBC Padding Oracle，伪造指定的 32 字节明文；再在最多 31 字节、末字节必须为 `0x00` 的限制下编写排序程序。

## 解题过程

### 1. 利用 Padding Oracle 构造任意明文

设 AES 分组解密结果为

$$
I_i=D_K(C_i),
$$

则 CBC 明文满足

$$
P_1=I_1\oplus IV,
$$

$$
P_2=I_2\oplus C_1.
$$

先固定目标密文块 $C_2$，把可控块 $X$ 放在它前面提交。服务端最后一个明文分组为 $D_K(C_2)\oplus X$。从末字节向前枚举 $X$：第 1 轮令末字节成为 `0x01`，第 2 轮令末两字节成为 `0x02 0x02`，依次进行，便可恢复完整的中间值 $I_2=D_K(C_2)$。

得到 $I_2$ 后，对期望的第二个明文块 $P_2$ 取

$$
C_1=I_2\oplus P_2.
$$

再以同样的方法恢复 $I_1=D_K(C_1)$，最后计算

$$
IV=I_1\oplus P_1.
$$

提交 `ticket = C1 || C2` 与 `fortune = IV`，解密结果就是指定的两个明文块。

下面给出与远端协议无关的完整核心实现。传入的 `oracle(ticket, fortune)` 只需在同一连接中完成一次交互，并在回显不是 `invaild ticket` 时返回 `True`：

```python
def xor_bytes(left, right):
    if len(left) != len(right):
        raise ValueError("length mismatch")
    return bytes(a ^ b for a, b in zip(left, right))


def recover_intermediate(target_block, oracle):
    """利用 padding oracle 恢复 D_K(target_block)。"""
    if len(target_block) != 16:
        raise ValueError("AES block must contain 16 bytes")
    if b"\n" in target_block:
        raise ValueError("target block cannot pass the line-based input")

    intermediate = bytearray(16)
    probe = bytearray(16)
    zero_iv = bytes(16)

    for position in range(15, -1, -1):
        padding = 16 - position
        for index in range(position + 1, 16):
            probe[index] = intermediate[index] ^ padding

        found = False
        for candidate in range(256):
            probe[position] = candidate
            ticket = bytes(probe) + target_block
            if b"\n" in ticket or not oracle(ticket, zero_iv):
                continue

            # 最后一字节可能误撞原有的多字节合法填充，翻转其前一字节复核。
            if position == 15:
                confirmation_ticket = None
                for delta in range(1, 256):
                    confirmation = bytearray(probe)
                    confirmation[14] ^= delta
                    trial = bytes(confirmation) + target_block
                    if b"\n" not in trial:
                        confirmation_ticket = trial
                        break
                if confirmation_ticket is None:
                    raise RuntimeError("cannot build a line-safe confirmation")
                if not oracle(confirmation_ticket, zero_iv):
                    continue

            intermediate[position] = candidate ^ padding
            found = True
            break

        if not found:
            raise RuntimeError(f"no valid byte at position {position}")

    return bytes(intermediate)


def forge_ticket(plaintext, oracle, c2=None):
    if len(plaintext) != 32:
        raise ValueError("this construction requires exactly two blocks")

    p1, p2 = plaintext[:16], plaintext[16:]
    if c2 is None:
        c2 = bytes(16)
    if len(c2) != 16 or b"\n" in c2:
        raise ValueError("C2 must be a 16-byte line-safe block")
    i2 = recover_intermediate(c2, oracle)
    c1 = xor_bytes(i2, p2)
    i1 = recover_intermediate(c1, oracle)
    iv = xor_bytes(i1, p1)

    ticket = c1 + c2
    if b"\n" in ticket or b"\n" in iv:
        raise RuntimeError("choose another C2 and retry to avoid newline bytes")
    return ticket, iv
```

程序使用按行读取，遇到字节 `0x0a` 就提前结束，因此探测块、最终密文和 IV 都不能含换行。上面的实现会跳过探测中的换行并检查最终结果；若由固定 $C_2$ 推导出的 $C_1$ 或 IV 含换行，应更换 $C_2$ 后重新构造，不能把截断误认为 oracle 结果。

### 2. 构造受限 VM 排序程序

VM 有 7 个寄存器：`PC`、`IDX1`、`IDX2`、`TMP1`、`TMP2`、`IMM` 和 `FLAG`。指令按 4 位对齐，表中的长度单位是位而不是字节：

| 操作码 | 操作数 | 语义 | 长度 |
| --- | --- | --- | ---: |
| `0` | 无 | 检查数组是否有序 | 4 位 |
| `1` | `reg` | `reg += IMM` | 8 位 |
| `2` | `reg` | `reg -= IMM` | 8 位 |
| `3` | `tmp, idx` | `tmp = list[idx]` | 12 位 |
| `4` | `idx, tmp` | `list[idx] = tmp` | 12 位 |
| `5` | `imm` | `IMM = imm` | 12 位 |
| `6` | `imm` | 根据 `FLAG` 条件设置 `IMM` | 12 位 |
| `7` | 无 | 比较 `list[IDX1]` 与 `list[IDX2]` | 4 位 |
| `8` | `idx` | 比较索引寄存器与 `IMM` | 8 位 |
| `9` | `idx` | `IMM = idx` | 8 位 |

可用两个索引实现双重循环：`IDX1` 指向当前基准元素，`IDX2` 从 `IDX1 + 1` 递增到 31，比较每一对元素并在顺序错误时交换；一轮结束后增加 `IDX1`，重置 `IDX2`，最后执行 `check`。为了压缩长度，payload 先无条件交换两个元素，再根据比较结果通过相对跳转决定保留交换还是跳回同一段代码再交换一次以撤销；循环复位也复用 `IMM` 与条件赋值，避免单独的跳转指令。

最终 VM 程序为 31 字节，十六进制表示如下：

```text
51f822566810501123323414244137598654209222815011191125f46042000
```

其有效指令占 30.5 字节，最后半字节补零后，整个程序的末字节正好为 `0x00`。AES 解密后的明文还必须满足 PKCS#7，因此在这 31 字节后追加一个 `0x01`，得到完整的 32 字节目标明文：

```text
51f822566810501123323414244137598654209222815011191125f460420001
```

把该字节串传给前面的 `forge_ticket()`，再提交返回的密文与 IV。服务端去掉末尾 `0x01` 后，VM 收到的正是以 `0x00` 结束的 31 字节程序，排序检查通过后即可读取 flag。

## 方法总结

这道题把密码利用与受限程序设计串在一起。第一阶段的核心不是破解 AES，而是利用“填充错误”和“填充正确后继续执行”两条可区分路径，逐字节恢复分组解密中间值；第二阶段则要把双重循环、条件交换和跳转复用压缩到 31 字节以内。

最容易混淆的是三种长度：VM 有效指令占 30.5 字节，补齐后程序占 31 字节且以 `0x00` 结束，最后再追加 `0x01` 才形成两个完整 AES 分组。另一个实现陷阱是按行协议中的 `0x0a`，它不仅影响明文，也会截断 oracle 探测所用的密文或 IV。
