# SycOS

## 题目简述

附件基于 xv6 RISC-V 2020 labs，新增系统调用 22 `sycgame(addrA, addrB)`。它取得两个用户虚拟地址对应的 PTE，直接交换两个页表项并执行 `sfence_vma()`，因此交换的是整页映射，而不是两个指针或两段缓冲区内容。

用户程序要求 64 字节输入。每个字符作为自定义 LCG 的种子生成 128 字节，前 32 个字符组成 `data1`，后 32 个组成 `data2`。随后执行 16 轮：对两页分别做两种 TEA 风格变换、交换一对 256 字节分块、调用 `sycgame` 交换整页映射，最后与两个 4096 字节常量比较。

## 解题过程

### 明确一轮的真实顺序

源码一轮操作是：

```text
data1 = enc1_all(data1)
data2 = enc2_all(data2)
swap(data1[i*256:(i+1)*256], data2[(15-i)*256:(16-i)*256])
swap_page_mapping(data1, data2)
```

`sycgame` 的关键代码等价于：

```c
pteA = walk(pagetable, addrA, 0);
pteB = walk(pagetable, addrB, 0);
tmp = *pteA;
*pteA = *pteB;
*pteB = tmp;
sfence_vma();
```

离线求逆时可把 PTE 交换建模成交换两个完整的 4096 字节数组。页交换和 256 字节分块交换都是自身的逆操作，所以必须从第 15 轮倒序执行：

```python
for round_index in range(15, -1, -1):
    data1, data2 = data2, data1
    swap_256_byte_chunks(data1, round_index, data2, 15 - round_index)
    decrypt_all_enc2_blocks(data2)
    decrypt_all_enc1_blocks(data1)
```

顺序不能写成“先解 TEA 再换页”。对复合函数 $P\circ S_i\circ E$，逆函数是 $E^{-1}\circ S_i^{-1}\circ P^{-1}$，实现时首先执行最后发生的整页交换。

### 逆两种 TEA 风格变换

`enc1` 是 16 轮标准方向的 TEA 结构，最后的 `sum` 为
$16\cdot0x9e3779b9\bmod2^{32}=0xe3779b90$。求逆时先减 `v1`，再减 `v0`，每轮最后减 `delta`。

`enc2` 从 `sum=0` 开始，每轮依次对 `v1`、`v0` 做减法，再增加 `delta`，共 8 轮。逆运算从
$8\cdot delta\bmod2^{32}=0xf1bbcdc8$ 开始，先减 `sum`，再按相反次序给 `v0`、`v1` 加回原表达式。所有中间量都要截断为 32 位，并按附件运行平台的小端序拆分每个 8 字节块。

```python
MASK = 0xffffffff
DELTA = 0x9e3779b9
KEY = (0x11222233, 0xaabbccdd, 0x1a2b3c4d, 0xcc1122aa)

def dec1(v0, v1):
    total = (16 * DELTA) & MASK
    for _ in range(16):
        v1 = (v1 - (((v0 << 4) + KEY[2]) ^ (v0 + total) ^ ((v0 >> 5) + KEY[3]))) & MASK
        v0 = (v0 - (((v1 << 4) + KEY[0]) ^ (v1 + total) ^ ((v1 >> 5) + KEY[1]))) & MASK
        total = (total - DELTA) & MASK
    return v0, v1

def dec2(v0, v1):
    total = (8 * DELTA) & MASK
    for _ in range(8):
        total = (total - DELTA) & MASK
        v0 = (v0 + (((v1 << 4) + KEY[0]) ^ (v1 + total) ^ ((v1 >> 5) + KEY[1]))) & MASK
        v1 = (v1 + (((v0 << 4) + KEY[2]) ^ (v0 + total) ^ ((v0 >> 5) + KEY[3]))) & MASK
    return v0, v1
```

### 逐字符反查 LCG 数据块

逆完 16 轮后，每 128 字节只依赖一个输入字符。前后两页都使用局部下标 `i=0..31`，种子为 `ord(flag[i]) + i` 或 `ord(flag[32+i]) + i`。枚举可打印字符、生成 128 字节并比较即可：

```python
def lcg_block(seed):
    state = seed
    out = bytearray()
    for _ in range(128):
        state = (state * 1103515245 + 12345) & 0xffffffffffffffff
        out.append(((state // 65536) % 32768) & 0xff)
    return bytes(out)

def recover_page(page):
    result = bytearray()
    for index in range(32):
        target = page[index * 128:(index + 1) * 128]
        match = next(
            value for value in range(0x20, 0x7f)
            if lcg_block(value + index) == target
        )
        result.append(match)
    return bytes(result)

flag = recover_page(data1) + recover_page(data2)
```

按源码的 `unsigned long` 宽度模拟 LCG 后，得到：

```text
SCTF{xv6_nice_lab_6666_YOU_ARE_6666666_OrzzzzzzzzzzzzzrO_wowowo}
```

把结果重新走正向 16 轮后，两页应分别与 `data1_cmp`、`data2_cmp` 逐字节相同，这是比只依赖源码注释更强的验证。

## 方法总结

题目的迷惑点在于把普通缓冲区变换与操作系统页表操作混在一起。`sycgame` 交换 PTE，所以离线模型必须在每轮交换整页；逆复合变换时还要严格倒序处理页交换、分块交换、`enc2`、`enc1`。恢复到原始页后，每个字符只控制独立的 128 字节 LCG 块，逐字符穷举即可。分析系统题时，先明确系统调用改变的是数据、指针还是地址映射，往往比直接啃大常量更重要。
