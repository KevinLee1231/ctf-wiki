# ancient

## 题目简述

本题是基于算术编码的逆向题。程序先校验 56 字节 flag 长度，并用 FNV-1a 检查前缀 `d3ctf`；随后把用户输入拼接到固定字符串后做动态算术编码，再与受编译期字符串保护的数据逐位比较。

题目附件包含源码和二进制，核心是一个带 FNV-1a 前缀检查的算术编码校验程序。解题主线是识别算术编码实现和固定分布表：虽然函数名和流程表现为动态编码表，但对 flag 编码时实际使用由固定字符串导出的稳定概率分布，因此可以 dump 目标编码再解码，或逐字符调试爆破。

## 解题过程

题目源码见 [`pcy190/d3ctf-ancient`](https://github.com/pcy190/d3ctf-ancient)。下文已经把源码中的固定字符串、算术编码模型、目标编码数据和两种求解方式转写出来，避免后续依赖仓库才能理解主线。

这题的核心是算术编码。算术编码不是逐字符替换，而是根据概率分布不断收窄 `[low, high)` 区间，整段消息最终对应区间中的一个数；解码时只要使用完全相同的概率表和更新规则，就能从目标编码值反推出原文。算法模型可对照 Amir Said 的 HP Labs 报告《Introduction to Arithmetic Coding - Theory and Practice》<https://www.hpl.hp.com/techreports/2004/HPL-2004-76.pdf>、实现参考 <https://github.com/nibrunie/ArithmeticCoding> 以及中文介绍 <https://zhuanlan.zhihu.com/p/23834589>。

程序首先检查 flag 长度为 56 字节，并把前五个字符的 FNV-1a 哈希与 `d3ctf` 的哈希比较。随后程序把输入拼接到固定字符串后调用 `encode_value_with_update`：

```text
This is an ancient string, it represents the origin of all binary characters,
isn't it. Let me see, it says 0,1,2,3,4,5,6,7...
```

表面上这里使用动态编码表，编码分布和传入字符串有关；但实际对 flag 编码时，概率分布由前面的固定字符串稳定导出。也就是说，虽然函数名里带 `update`，对未知 flag 部分仍可视为使用同一张静态表。

题目中字符串都做了编译期保护：模板类把字符串与同一个 key 异或后存储，包括最终比较用的目标编码数据。这个 key 可以静态还原，也可以在动态调试时 dump。程序最后逐位比较编码结果，178 个有效字符匹配即通过，这个长度来自固定字符串 126 字节加 56 字节输入编码后的结果，再去掉末尾 4 个无用编码 0。

源码里还放了一个 glibc 2.33-4 版本说明字符串，末尾包含 Arch Linux 的报 bug 地址 <https://bugs.archlinux.org/>。它只是版本指纹/干扰字符串，不参与 flag 长度、FNV-1a 或算术编码校验。

`loop_for_junk(loc_401820)` 只是反符号执行干扰。它包含花指令和多轮循环，但传入的 index 指针在执行后没有被实际修改，可以 patch 掉，也可以在动态爆破时直接忽略。

解法有两条：

1. dump 出解密后的目标编码数据，把固定字符串生成的概率表和目标编码传入同一套算术解码逻辑，直接得到 flag。
2. 逐字符爆破。在比较点 `0x401C83` 下断点，观察 `rbp+var_14` 对应的 index 是否比上一轮增加；如果增加，说明当前新增字符正确。

PDF 中给出了动态 dump 后的目标编码数据。为了在删除原 PDF 后仍能复现解码，可把它保存为字节串；前 126 字节对应固定字符串编码后的部分，后面是 flag 编码数据和末尾填充：

```python
target = bytes.fromhex(
    "5467697320697320616e20616e6369656e7420737472696e672c206974207265"
    "70726573656e747320746865206f726967696e206f6620616c6c2062696e6172"
    "7920636861726163746572732c2069736e27742069742e204c6574206d652073"
    "65652c206974207361797320302c312c322c332c342c352c362c372e2e2e67f3"
    "a3ca2358a3d1f8c196e3d78585febe7bd28259f4d8f05ff5e255e52c14dcd6f4"
    "60f989840c7050b8f5de7fff5ac88d61f002000000000000"
)
```

解码思路可以整理为以下骨架：

```cpp
void writeup() {
    auto *decomp = static_cast<unsigned char *>(malloc(local_size * 2));
    ac_state_t state;
    init_state(&state, 16);
    reset_uniform_probability(&state);
    decode_value_with_update(
        decomp,
        target,
        &state,
        local_size,
        strlen(MAGIC_STRING),
        0
    );
    cout << decomp << endl;
}
```

## 方法总结

- 核心技巧：识别算术编码并判断概率表是否真正依赖输入；当固定前缀决定分布表时，可以把目标编码数据 dump 出来再走解码流程。
- 识别信号：固定长字符串、`encode_value_with_update` / `decode_value_with_update` 这类编码函数、逐位比较编码结果，以及用花指令循环干扰符号执行。
- 复用要点：字符串保护和 junk loop 只是外层干扰；优先确认编码算法和比较数据来源。若不能直接静态还原，可在比较点观察 index 变化逐字符爆破。

