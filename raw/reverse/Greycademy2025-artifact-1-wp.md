# Artifact 1

## 题目简述

附件是 stripped、PIE 的 x86-64 ELF，程序读取一个整数并与内部常量比较；正确分支会直接打印 flag。这是一道基础反编译与字符串检索训练，目标是从调用链恢复比较值和成功分支行为。

## 解题过程

反编译 `main` 后，可以把四个未命名函数依次理解为“读取输入”“返回检查值”“比较”“正确/错误处理”：

```c
guess = get_input_from_player();
check = get_value_of_1337();
if (check_equal(guess, check))
    correct();
else
    wrong();
```

检查返回常量的函数可知 `check == 1337`。输入该值即可进入成功分支：

```text
Enter your guess: 1337
Correct!
Here is your flag: grey{i_am_a_reverse_engineer!}
```

本题还有更短的字符串路线，因为 flag 作为明文常量编进 ELF：

```bash
strings artifact-1.elf | grep 'grey{'
```

两条路线得到相同结果。官方解答中的五张 IDA 图片只是上述伪代码、字符串窗口和菜单操作的截图，信息已完整转写，因此没有保留图片。

## 方法总结

逆向新样本时应先跑 `strings`，再从 `main` 的输入、比较和成功分支建立最小调用链。重命名函数和变量能降低反编译噪声，但反编译器猜错参数数量并不一定影响真实控制流；应以调用关系和底层行为为准。
