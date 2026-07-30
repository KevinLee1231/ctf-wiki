# SU_ezlua

## 题目简述

附件包含一个定制 Lua 5.1.5 解释器 `lua` 和预编译字节码 `chall.luac`。Lua 指令集本身没有修改，但 chunk 的指令、数值常量和字符串采用了自定义序列化；即使恢复成标准 luac 并反编译，直接按脚本编写逆算法仍会失败，因为解释器还暗改了 `string.byte`。

完整解法分三层：

1. 对照 Lua 5.1 的 `luaU_undump`，把定制 chunk 转回标准格式；
2. 反编译脚本并恢复修改版 RC4、动态 delta 的 TEA 变体；
3. 逆向解释器中的 `string.byte`，把隐藏的逐字节变换纳入解密。

## 解题过程

解释器启动时注册 `And`、`Or`、`Xor`、`Shl`、`Shr`，随后把 `chall.luac` 作为固定脚本交给 `lua_cpcall`。根据标准 Lua 5.1 源码，chunk 加载入口是 `luaU_undump`；用签名常量 `0x61754c1b`，即字节串 `\x1bLua`，可以在二进制中定位它。

与标准加载器对比可发现三处变化。

第一，`LoadCode` 对每个四字节指令执行：

```c
decoded = rol32(bswap32(encoded ^ 0x32547698), 15);
```

第二，数值常量的序列化类型标记由标准的 `3` 改为 `0x3f`，数据也不再是八字节 double，而是四字节编码整数。加载时先做与指令相同的变换，再把有符号 32 位整数转换成 Lua number。

第三，`LoadString` 在读入字符串后，把每个字节循环左移 3 位：

```c
decoded_byte = rol8(encoded_byte, 3);
```

下面给出完整的递归转换器。它会同时处理嵌套 prototype、调试字符串和定制数值常量：

```python
#!/usr/bin/env python3
from pathlib import Path
from struct import pack

KEY = 0x32547698

def rol8(value, bits):
    return ((value << bits) | (value >> (8 - bits))) & 0xff

def rol32(value, bits):
    return ((value << bits) | (value >> (32 - bits))) & 0xffffffff

def bswap32(value):
    return int.from_bytes(value.to_bytes(4, "little"), "big")

def decode_word(value):
    return rol32(bswap32(value ^ KEY), 15)

class Reader:
    def __init__(self, data):
        self.data = data
        self.pos = 0

    def take(self, size):
        value = self.data[self.pos:self.pos + size]
        if len(value) != size:
            raise EOFError((self.pos, size))
        self.pos += size
        return value

    def u8(self):
        return self.take(1)[0]

    def u32(self):
        return int.from_bytes(self.take(4), "little")

    def u64(self):
        return int.from_bytes(self.take(8), "little")

def convert_string(reader):
    size = reader.u64()
    out = bytearray(pack("<Q", size))
    if size:
        out.extend(rol8(byte, 3) for byte in reader.take(size))
    return out

def convert_function(reader):
    out = bytearray()

    # source、linedefined、lastlinedefined、
    # nups、numparams、is_vararg、maxstacksize
    out += convert_string(reader)
    out += reader.take(8)
    out += reader.take(4)

    code_count = reader.u32()
    out += pack("<I", code_count)
    for _ in range(code_count):
        out += pack("<I", decode_word(reader.u32()))

    constant_count = reader.u32()
    out += pack("<I", constant_count)
    for _ in range(constant_count):
        custom_type = reader.u8()
        standard_type = 3 if custom_type == 0x3f else custom_type
        out += bytes([standard_type])

        if custom_type == 0:       # nil
            pass
        elif custom_type == 1:     # boolean
            out += reader.take(1)
        elif custom_type == 0x3f:  # custom number
            number = decode_word(reader.u32())
            if number & 0x80000000:
                number -= 1 << 32
            out += pack("<d", float(number))
        elif custom_type == 4:     # string
            out += convert_string(reader)
        else:
            raise ValueError(("constant type", custom_type, reader.pos))

    prototype_count = reader.u32()
    out += pack("<I", prototype_count)
    for _ in range(prototype_count):
        out += convert_function(reader)

    line_count = reader.u32()
    out += pack("<I", line_count)
    out += reader.take(4 * line_count)

    local_count = reader.u32()
    out += pack("<I", local_count)
    for _ in range(local_count):
        out += convert_string(reader)
        out += reader.take(8)

    upvalue_count = reader.u32()
    out += pack("<I", upvalue_count)
    for _ in range(upvalue_count):
        out += convert_string(reader)

    return out

source = Path("chall.luac").read_bytes()
reader = Reader(source)
header = reader.take(12)
assert header == b"\x1bLua\x51\x00\x01\x04\x08\x04\x08\x00"

converted = header + convert_function(reader)
assert reader.pos == len(source)
Path("out.luac").write_bytes(converted)
print(len(source), len(converted))
```

对附件运行时，输入长度为 3491，转换结果为 3655 字节，解析游标恰好消费全部输入。`out.luac` 可交给兼容 Lua 5.1 的 unluac 或 luadec；两者输出相互对照后，核心校验流程为：

```lua
key = "thisshouldbeakey"
data = rc4(flag_body, key)

for block = 0, 3 do
    v0 = to_uint(data, 1 + 8 * block)
    v1 = to_uint(data, 5 + 8 * block)
    v0, v1 = encrypt(v0, v1, key)
    result = result .. from_uint(v0) .. from_uint(v1)
end

return hex(result) ==
  "ac0c0027f0e4032acf7bd2c37b252a93" ..
  "3091a06aeebc072c980fa62c24f486c6"
```

输入总长必须为 39，格式为 `SUCTF{`、32 字节正文、`}`。

反编译出的 RC4 也不是标准实现。KSA 仍然置换 256 字节 S 盒，但 PRGA 使用：

```text
keystream = S[(S[i] - S[j]) & 0xff]
j = (j + ciphertext_byte) & 0xff
```

TEA 部分把 `thisshouldbeakey` 解释为四个 32 位 key word，初始 delta 为 `0x12345678`。每轮开始前，delta 都会再经过一次上述修改版 RC4，然后累加到 sum：

```text
delta = to_uint(modified_rc4(from_uint(delta), key))
sum = (sum + delta) & 0xffffffff
```

随后执行 32 轮 TEA 风格更新：

```text
v0 += ((v1 << 4) + k0) ^ (v1 + sum) ^ ((v1 >> 5) + k1)
v1 += ((v0 << 4) + k2) ^ (v0 + sum) ^ ((v0 >> 5) + k3)
```

仅按这些脚本逻辑写逆算法仍得不到正确结果。把解释器 main 中传给 `lua_cpcall` 的脚本参数改为 `NULL` 后，可以进入交互模式。标准 Lua 中：

```lua
string.byte("0", 1)
```

应返回 48，题目解释器却返回 83。追踪 string 库注册表中的 `byte` 实现，得到对第 $i$ 个零基字节的真实变换：

$$
\operatorname{string\_byte}(s,i)
=((s_i\oplus i)+0x23)\bmod 256.
$$

逆变换为：

$$
s_i=((v_i-0x23)\bmod 256)\oplus i.
$$

这个函数会影响 RC4 的 key、明文、`to_uint()` 和最后的 `hex()`，所以必须贯穿整个逆过程。下面是与仓库 `exp.c` 等价、已验证可直接运行的 Python 解密器：

```python
from struct import pack, unpack

MASK = 0xffffffff
KEY = b"thisshouldbeakey"

def string_byte(data, index):
    return ((data[index] ^ index) + 0x23) & 0xff

def reverse_string_byte(data, start=0, size=None):
    if size is None:
        size = len(data) - start
    for i in range(start, start + size):
        data[i] = ((data[i] - 0x23) & 0xff) ^ i

def to_uint(data, offset=0):
    return sum(
        string_byte(data, offset + i) << (8 * i)
        for i in range(4)
    )

def make_box(key):
    box = list(range(256))
    expanded = [
        string_byte(key, i % len(key))
        for i in range(256)
    ]
    j = 0
    for i in range(256):
        j = (j + box[i] + expanded[i]) & 0xff
        box[i], box[j] = box[j], box[i]
    return box

def rc4_encrypt(data, key):
    box = make_box(key)
    i = j = 0
    for k in range(len(data)):
        i = (i + 1) & 0xff
        j = (j + box[i]) & 0xff
        box[i], box[j] = box[j], box[i]
        value = (
            string_byte(data, k)
            ^ box[(box[i] - box[j]) & 0xff]
        )
        data[k] = value
        j = (j + value) & 0xff

def rc4_decrypt(data, key):
    box = make_box(key)
    i = j = 0
    for k in range(len(data)):
        i = (i + 1) & 0xff
        j = (j + box[i]) & 0xff
        box[i], box[j] = box[j], box[i]
        cipher_byte = data[k]
        data[k] ^= box[(box[i] - box[j]) & 0xff]
        j = (j + cipher_byte) & 0xff
    reverse_string_byte(data)

def previous_delta(delta):
    raw = bytearray(pack("<I", delta))
    reverse_string_byte(raw)
    rc4_decrypt(raw, KEY)
    return unpack("<I", raw)[0]

def tea_decrypt(v0, v1):
    words = [to_uint(KEY, 4 * i) for i in range(4)]
    delta = 0x12345678
    total = 0

    # 先重放正向 delta 链，得到最后一轮的 sum 和 delta。
    for _ in range(32):
        raw = bytearray(pack("<I", delta))
        rc4_encrypt(raw, KEY)
        delta = to_uint(raw)
        total = (total + delta) & MASK

    for _ in range(32):
        v1 = (
            v1
            - (((v0 << 4) + words[2])
               ^ (v0 + total)
               ^ ((v0 >> 5) + words[3]))
        ) & MASK
        v0 = (
            v0
            - (((v1 << 4) + words[0])
               ^ (v1 + total)
               ^ ((v1 >> 5) + words[1]))
        ) & MASK
        total = (total - delta) & MASK
        delta = previous_delta(delta)

    return v0, v1

encrypted = bytearray.fromhex(
    "ac0c0027f0e4032acf7bd2c37b252a93"
    "3091a06aeebc072c980fa62c24f486c6"
)

# 逆 hex() 内部的定制 string.byte。
reverse_string_byte(encrypted)

for offset in range(0, 32, 8):
    left, right = unpack("<II", encrypted[offset:offset + 8])
    left, right = tea_decrypt(left, right)
    encrypted[offset:offset + 8] = pack("<II", left, right)

# 逆 to_uint() 层，再逆最外层修改版 RC4。
reverse_string_byte(encrypted)
rc4_decrypt(encrypted, KEY)

print(f"SUCTF{{{encrypted.decode()}}}")
```

输出：

```text
SUCTF{341528c2bde511efade200155d8503ef}
```

仓库中的 `exp.c` 在本地重新编译运行后也得到同一结果。

## 方法总结

本题有两个容易混淆的“魔改层”：luac 序列化只影响静态分析工具能否读懂字节码，解释器中的 `string.byte` 则真正改变题目算法。恢复出看似合理的 Lua 源码后仍解不出结果，说明不能立刻怀疑 TEA 逆序或端序，应把脚本依赖的宿主函数逐个放进交互解释器做差分测试。

转换 chunk 时也不能只批量替换字节：数值常量由四字节整数扩展成八字节 double，会改变后续所有字段位置，必须递归解析完整 Lua 5.1 prototype。最终解密则要明确每次 `string.byte` 出现的位置，按“外层 hex 逆变换、TEA 逆变换、`to_uint` 逆变换、RC4 逆变换”的反向顺序处理。
