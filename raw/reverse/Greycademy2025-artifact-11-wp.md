# Greycademy2025 Artifact 11: Decompilation

## 题目简述

附件是一个未剥离符号的 ELF，用于练习从反编译结果恢复输入变换。程序会修改输入的首尾字符、追加固定后缀，再与目标句子比较；通过后才解码 flag。

## 解题过程

沿 `main` 的调用链可见三步处理：

```c
input[0] -= 0x20;
input[strlen(input) - 1] -= 16;
strcat(input, "emy! <3");
```

最终比较目标是：

```text
I love greycademy! <3
```

去掉追加的 `emy! <3` 后，变换结果应为 `I love greycad`。反向恢复首字符：`'I' + 0x20 = 'i'`；恢复尾字符：`'d' + 16 = 't'`。因此原始输入是：

```text
i love greycat
```

把它交给二进制，三步变换依次得到：

```text
i -> I
t -> d
I love greycad -> I love greycademy! <3
```

比较通过后，`correct` 用固定种子 67 的 `rand()`、相邻字节和内置 `KEY` 对 21 个字节逐项异或。无需单独逆这个阶段，因为程序会在正确输入上直接执行并打印：

```text
grey{cr4ck3d_1t_0p3n}
```

## 方法总结

逆向应优先从最终比较点向前做数据流恢复。这里所有变换都是可逆的：固定后缀可移除，字符减法可加回。flag 解码函数虽然更复杂，却位于成功分支之后；先恢复门控输入并让原程序执行它，比无必要地重写整套 PRNG/XOR 逻辑更简洁，也更贴合真实控制流。
