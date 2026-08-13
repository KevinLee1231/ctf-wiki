# Greycademy2025 Artifact 14: Pointers

## 题目简述

程序构造三层指针表 `src -> inter -> dst`，其中 `dst` 是小写字母表再加下划线和一对花括号的字符集合。检查函数用大量强制类型转换和指针加减，为输入的 31 个位置各选出一个目标字符。

## 解题过程

初始化关系很简单：

```c
char dst[] = "abcdefghijklmnopqrstuvwxyz_{}";
char *inter[29];
char **src[29];

inter[i] = &dst[i];
src[i] = &inter[i];
```

复杂表达式的障眼法在于 C 指针算术会按目标类型缩放。例如 `(short *)ptr + 8` 前进 16 字节，而 `(char *)ptr + 16` 前进同样的字节数；转换类型本身不改变地址。每条约束都经过四次这种地址移动，中间一次解引用回到 `inter`，最后再解引用到 `dst` 的某个字符。

可将表达式从内向外规约为字节偏移。对任意 `(T *)p + n`，地址变化都是 `n * sizeof(T)`；遇到 `*` 时，根据 `src[i] = &inter[i]` 和 `inter[i] = &dst[i]` 换到下一层表。逐条计算后，31 个目标字符依次为：

```text
grey{puzzle_and_pointer_expert}
```

这个 flag 本身恰好长 31 字节。`fgets(inp, 32, stdin)` 最多接收 31 个字符，因此输入完 flag 后按下回车时，换行仍留在输入流中，并不会进入 `inp`；这也解释了 `strlen(inp) == 31`。直接验证：

```bash
printf 'grey{puzzle_and_pointer_expert}\n' | ./artifact-14
```

输出：

```text
Correct!
```

flag 本体为：

```text
grey{puzzle_and_pointer_expert}
```

## 方法总结

强制转换改变的是编译器解释步长和解引用类型，不会凭空移动数据。处理嵌套指针题时，把每种类型的 `sizeof` 列成表，所有加减先统一换算成字节偏移，再在真正的 `*` 处切换指针层级。还要沿输入函数检查结尾行为：本题的 31 字节上限让 `fgets` 在读入换行前就填满缓冲区，flag 与换行不会混为一体。
