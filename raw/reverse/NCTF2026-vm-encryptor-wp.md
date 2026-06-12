# VM Encryptor

## 题目简述

自定义 VM 逆向题。先还原 VM 结构体和 opcode，写反汇编器恢复逻辑；核心算法是魔改 Base64：每 3 字节拼成 24 位后做多轮循环位移与异或，再映射 Base64 表，最后整体异或 `0x63`。

## 解题过程

首先还原一下vm的结构体：

```c
enum ErrorCode {
    ERR_NONE = 0,
    ERR_INVALID_OPCODE,
    ERR_STACK_OVERFLOW,
    ERR_STACK_UNDERFLOW,
    ERR_MEMORY_ACCESS,
    ERR_DIVIDE_BY_ZERO,
    ERR_IO,
};
struct vm {
    uint32_t pc;
    uint32_t sp;
    uint32_t bp;
    uint8_t mem[0x10000];
    uint32_t stack[0x1000];
    uint32_t running;
    enum ErrorCode error;
};
```

然后标注指令集：

```c
enum Opcode {
    JP = 0,
    JZ,
    JNZ,
    PUSH_BYTE,
    PUSH_DWORD,
    PUSH_BYTE_PTR,
    PUSH_DWORD_PTR,
    POP,
    POP_BYTE,
    POP_DWORD,
    ADD,
    SUB,
    MUL,
    DIV,
    MOD,
    AND,
    OR,
    NOT,
    XOR,
    SHL,
    SHR,
    CMP,
    EQ,
    NE,
    LT,
    GT,
    LE,
    GE,
    DUP,
    SWAP,
    CALL,
    RET,
    PUT,
    HALT = 0xFF,
};
```

拿去给AI，让AI写一个反汇编器出来，大概长这样：

```text
.text
loc_0000:
    PUSH_DWORD var_1000
    PUSH_DWORD var_102B
    PUSH_DWORD 0x0000002A
    CALL loc_004F
    PUSH_DWORD var_102B
    PUSH_DWORD var_1064
    PUSH_DWORD 0x00000038
    CALL loc_0368
    PUSH_DWORD var_1064
    PUSH_DWORD var_109D
    PUSH_DWORD 0x00000038
    CALL loc_0471
    JZ loc_0048
    PUSH_DWORD var_10D5
    PUT
    HALT
loc_0048:
    PUSH_DWORD var_10DC
    PUT
    HALT
loc_004F:
    PUSH_DWORD 0x00000000
    PUSH_DWORD var_1147
    POP_DWORD
    JP loc_005F
loc_005F:
    PUSH_DWORD var_1147
    PUSH_DWORD_PTR
    PUSH_DWORD 0x00000000
    CMP
```

逻辑：先做一轮魔改 Base64 编码，再做按字节异或。魔改部分会把每组 3 字节拼成 24 位后进行三轮循环左移（5/11/20 位）并异或 0x55757D ，再映射到 Base64 表。最后再来一轮异或 0x63

```text
ROUND_SHIFTS = (5, 11, 20)
ROUND_XOR = 0x55757D
XOR_KEY = 0x63
B64_TABLE = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
TARGET_BYTES = bytes(
    [ 39, 90, 11, 8, 10, 58, 9, 13, 48, 49, 76, 55, 2, 58, 18, 14, 84, 42, 76,
48, 44, 50, 17, 39, 6, 49, 86, 55, 18, 38, 76, 55, 40, 50, 91, 55, 85, 0, 72,
26, 2, 1, 17, 39, 22, 0, 76, 55, 36, 7, 17, 52, 1, 54, 91, 39,
    ]
)
def ror24(value: int, shift: int) -> int:
    shift %= 24
    mask = 0xFFFFFF
    return ((value >> shift) | (value << (24 - shift))) & mask
def decrypt_flag() -> str:
    b64_bytes = bytes(b ^ XOR_KEY for b in TARGET_BYTES)
    table_index = {ord(ch): idx for idx, ch in enumerate(B64_TABLE)}
    sextets = [table_index[b] for b in b64_bytes]
    if len(sextets) % 4 != 0:
        raise ValueError("encoded length is not multiple of 4")
    plain = bytearray()
    for i in range(0, len(sextets), 4):
        chunk = (
            (sextets[i] << 18)
            | (sextets[i + 1] << 12)
            | (sextets[i + 2] << 6)
            | sextets[i + 3]
        ) & 0xFFFFFF
        for shift in reversed(ROUND_SHIFTS):
            chunk = ror24(chunk ^ ROUND_XOR, shift)
        plain.extend([(chunk >> 16) & 0xFF, (chunk >> 8) & 0xFF, chunk & 0xFF])
    return plain.decode("ascii")
if __name__ == "__main__":
    print(decrypt_flag())
NCTF{1578be15-ad09-4859-9193-5d52585eb485}
```

## 方法总结

- 核心技巧：自定义 VM 指令还原后逆向魔改 Base64 与 XOR。
- 识别信号：VM 字节码先输出固定成功/失败字符串并包含 PUSH/CALL/PUT/HALT 等指令时，应先做反汇编再分析算法。
- 复用要点：把编码过程拆成可逆的 24 位分组运算，按相反顺序恢复明文。
