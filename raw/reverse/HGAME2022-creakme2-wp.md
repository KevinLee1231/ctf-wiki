# creakme2

## 题目简述

这是一个 Windows x64 Release 程序，主体是修改版 XTEA。出题人用 SEH（Structured Exception Handling）把每轮对 `sum` 的额外变换藏在 `__except` 处理块中；如果只看 IDA 生成的普通伪代码，会漏掉这一步并解出乱码。

## 解题过程

加密函数先按 XTEA 结构更新 `v0`，随后执行：

```c
sum += 0x9E3779B1;
a = 1 / (sum >> 31);
```

当 `sum` 的最高位为 `0` 时，`sum >> 31` 等于 `0`，整数除法触发 `EXCEPTION_INT_DIVIDE_BY_ZERO`。异常过滤器返回 `EXCEPTION_EXECUTE_HANDLER` 后，真正的处理块执行：

```c
sum ^= 0x01234567;
```

源码中还放置了一个整数溢出过滤器，但这一分支没有在实际加密流程中生效。IDA x64 可以在 `__try` 末尾的 `__except at loc_...` 注释处找到处理块，再沿异常过滤函数确认它比较的异常码是 `EXCEPTION_INT_DIVIDE_BY_ZERO`。

解密时顺序必须完全反转：先用当前 `sum` 撤销 `v1`，再根据最高位撤销异或，随后减去 `delta`，最后撤销 `v0`。所有运算都在模 $2^{32}$ 下进行。目标数组和密钥分别为：

```text
target = 457e62cf 9537896c 1f7e7f72 f7a073d8
         8e996868 40afaf99 0f990e34 196f4086
key    = 00000001 00000002 00000003 00000004
```

完整解密脚本如下：

```python
MASK = 0xffffffff
DELTA = 0x9e3779b1
KEY = [1, 2, 3, 4]

TARGET = [
    0x457e62cf,
    0x9537896c,
    0x1f7e7f72,
    0xf7a073d8,
    0x8e996868,
    0x40afaf99,
    0x0f990e34,
    0x196f4086,
]

def mix(value: int, total: int, key_word: int) -> int:
    shifted = ((value << 4) & MASK) ^ (value >> 5)
    return (((shifted + value) & MASK) ^ ((total + key_word) & MASK)) & MASK

def decipher(v0: int, v1: int) -> tuple[int, int]:
    total = 0xc78e4d05

    for _ in range(32):
        v1 = (v1 - mix(v0, total, KEY[(total >> 11) & 3])) & MASK

        # 撤销异常处理块中的变换；异或不改变最高位。
        if total >> 31 == 0:
            total ^= 0x01234567

        total = (total - DELTA) & MASK
        v0 = (v0 - mix(v1, total, KEY[total & 3])) & MASK

    return v0, v1

plain_words = []
for offset in range(0, len(TARGET), 2):
    plain_words.extend(decipher(TARGET[offset], TARGET[offset + 1]))

plain = b"".join(word.to_bytes(4, "little") for word in plain_words)
print(plain.rstrip(b"\x00").decode())
```

输出为：

```text
hgame{SEH_s0und5_50_1ntere5ting}
```

## 方法总结

SEH 不只是错误恢复机制，也可以作为控制流隐藏手段。x64 IDA 无法总是把异常处理块自然还原进伪代码，因此需要结合 `__try`/`__except` 注释、异常过滤器和汇编跳转补全语义。对修改版分组密码，只有把隐藏状态变换放回正确轮次与逆序位置，解密才会成立。
