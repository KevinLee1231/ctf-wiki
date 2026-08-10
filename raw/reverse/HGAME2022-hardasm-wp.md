# hardasm

## 题目简述

题目使用 AVX2 指令对 32 字节输入进行校验。加密区间包含大量 `vpaddb`、`vpsubb`、`vpxor`、`vpshufb` 和 `vpermd`，直接阅读反编译代码很困难。可以在 IDAPython 中模拟并逆序执行这些向量指令，也可以修改最终比较结果的打印逻辑，把逐字节比较掩码变成一个前缀 oracle，再逐位爆破 flag。

## 解题过程

先用 IDAPython 遍历加密区间并统计助记符，核心操作只有五种：

```python
ea = 0x140001081
opcodes = set()

while ea < 0x140007F17:
    opcodes.add(print_insn_mnem(ea))
    ea = next_head(ea)

print(opcodes)
# {'vpaddb', 'vpsubb', 'vpxor', 'vpshufb', 'vpermd'}
```

它们分别对应 32 个字节上的模 $2^8$ 加法、模 $2^8$ 减法、异或、每个 128 位 lane 内的字节置换，以及 32 位双字置换。若保存正向指令及操作数，逆序遍历时可按下表撤销：

| 正向指令 | 逆操作 |
| --- | --- |
| `vpaddb dst, ..., src` | 对 `dst` 逐字节减去 `src` |
| `vpsubb dst, ..., src` | 对 `dst` 逐字节加上 `src` |
| `vpxor dst, ..., src` | 再异或一次 `src` |
| `vpshufb` | 按控制掩码建立逆字节置换 |
| `vpermd` | 按索引向量建立逆双字置换 |

对寄存器快照正向模拟一次，确认结果与程序内密文一致；随后把 `ymm0` 替换为密文并逆序执行记录的操作，即可恢复明文。题目还有一条更直观的动态路线：最终比较会生成 32 字节掩码，匹配的字节为 `0xff`，不匹配的字节为 `0x00`。已知前缀 `hgame{` 的输入会得到连续 6 个 `0xff`，说明该掩码可以直接泄露正确前缀长度。

程序原本根据比较结果选择字符串 `success` 或 `error` 后调用 `printf`。将两处分支中的字符串地址都改为比较掩码 `[rsp+70h+var_50]`，让 `printf` 直接打印该缓冲区。由于 `0xff` 非零而第一个错误字节为 `0x00`，输出长度正好等于当前正确前缀长度。对修改后的程序可使用以下 Python 3 脚本逐位爆破：

```python
import subprocess

binary = r"D:\ctf\hardasm-patched.exe"
prefix = bytearray(b"hgame{")

for position in range(6, 31):
    for candidate in range(32, 127):
        guess = bytearray(b" " * 32)
        guess[:len(prefix)] = prefix
        guess[position] = candidate
        guess[-1] = ord("}")

        result = subprocess.run(
            [binary],
            input=bytes(guess),
            stdout=subprocess.PIPE,
            stderr=subprocess.DEVNULL,
            check=False,
        )

        if len(result.stdout) > position:
            prefix.append(candidate)
            print((prefix + b"}").decode())
            break
    else:
        raise RuntimeError(f"位置 {position} 未找到可打印字符")

print((prefix + b"}").decode())
```

最终得到：

```text
hgame{right_your_asm_is_good!!}
```

## 方法总结

SIMD 代码看似庞大，但指令集合很小，先抽象每条指令对 32 字节状态的影响就能恢复普通数据流。完整逆模拟适用于保留了所有置换关系的情况；本题比较掩码又额外形成了逐字节前缀 oracle，因此修改输出后爆破更省力。无论采用哪条路线，都应先验证向量运算是按字节、按 128 位 lane 还是按 32 位元素进行，避免把 `vpshufb` 与 `vpermd` 的粒度混淆。
