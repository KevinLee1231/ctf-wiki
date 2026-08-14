# CakeCTF 2022 nimrev Writeup

## 题目简述

题目给出一个由 Nim 编译的 ELF。程序读取一行输入，与内置字符串比较，正确时输出 `Correct!`。Nim 运行时会给反编译结果带来较多初始化和内存管理代码，但真正的校验只有一层逐字节异或。

公开源码中的期望数组为 24 个字节，每个字节与 `0xff` 异或后拼接成目标字符串。

## 解题过程

若从二进制开始分析，可以先搜索 `Correct!` 和 `Wrong...`，沿交叉引用回到 `NimMainModule` 中的条件分支。最终会看到输入字符串和一个运行时生成的字符串传给 Nim 的 `eqStrings`。在该调用前下断点并查看两个参数，也可以直接取得目标字符串。

从公开源码看，校验等价于：

```nim
encoded.map do (c: char) -> char:
  char(uint8(c).xor(0xff))
```

因此直接复现变换即可：

```python
encoded = [
    0xBC, 0x9E, 0x94, 0x9A, 0xBC, 0xAB, 0xB9, 0x84,
    0x8C, 0xCF, 0x92, 0xCC, 0x8B, 0xCE, 0x92, 0xCC,
    0x8C, 0xA0, 0x91, 0xCF, 0x8B, 0xA0, 0xBC, 0x82,
]

flag = bytes(value ^ 0xff for value in encoded)
print(flag.decode())
```

运行结果：

```text
CakeCTF{s0m3t1m3s_n0t_C}
```

将该字符串输入原程序即可进入 `Correct!` 分支。

## 方法总结

这是一道熟悉非 C/C++ 编译产物的热身题。Nim 二进制中的运行时符号很多，但不应尝试完整理解所有初始化流程；从成功/失败字符串反向追踪到最终比较点，能够快速把范围收窄到真实校验逻辑。

当比较点已经明确时，可以动态查看参数，也可以静态还原输入变换。两条路线都应以原程序的 `Correct!` 结果作为最终验证。
