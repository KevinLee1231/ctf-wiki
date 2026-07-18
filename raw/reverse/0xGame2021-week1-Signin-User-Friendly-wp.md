# week1Signin: User Friendly

## 题目简述

程序让用户猜一个由 `srand(time(NULL))` 和 `rand()%1001` 生成的数字，但成功分支中的 flag 以明文常量直接编译进二进制。静态查看字符串即可得到结果，无需预测随机数。

## 解题过程

用 IDA 打开程序后按 `Shift+F12` 查看字符串，或在反编译窗口定位数字比较成功的分支，可以直接看到：

```text
0xGame{we1c0me_2_Rever5e_egin44ring}
```

源码逻辑如下：

```c
srand(time(NULL));
number = rand() % 1001;
scanf("%d", &input_number);
if (number == input_number)
    printf("0xGame{we1c0me_2_Rever5e_egin44ring}");
```

若要动态验证，也可以在 `rand()` 返回后或比较前下断点，读取 `number` 的实际值再继续运行；由于种子取当前时间，每次进程得到的数值可能不同。

## 方法总结

本题的最短路径是检查字符串和成功分支，而不是先攻击 PRNG。逆向时应先做低成本静态信息收集；只有目标字符串被加密或动态生成时，才需要进一步预测随机数、Patch 分支或动态调试。
