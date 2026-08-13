# Artifact 5

## 题目简述

附件是 stripped、PIE 的 C++ x86-64 ELF。程序要求输入 9 字节会员编号：首字母必须为 `T`（大小写均可），中间 7 位必须是数字，前两位代表的年份不大于 26，末字母由加权校验和决定。题面要求识别这种四词身份编号制度，而不是从程序中直接取出 flag 字符串。

## 解题过程

清理 C++ I/O 模板噪声后，校验逻辑可写成：

```c
weights = {2, 7, 6, 5, 4, 3, 2};
finals = "abcdefghizj";
res = sum(digit[i] * weights[i]);
year = 10 * digit[0] + digit[1];
index = 10 - (((res + 4) * 4 % 44) / 4);
valid = year <= 26 && lower(last) == finals[index];
```

其中 `c | 0x20` 用于把 ASCII 大写字母归一到小写。由于 $((4x) \bmod 44)/4 = x \bmod 11$，索引还可以写成 $10-((res+4) \bmod 11)$。

用全零数字验证模型：`T0000000` 的 `res=0`，加 4 后索引为 6，`finals[6] == 'g'`，所以 `T0000000G` 能使原程序输出 `Welcome back!`。

这种 `T` 开头、7 位数字加校验字母的结构对应 Singapore National Registration Identity Card。按题面要求用小写下划线封装：

```text
grey{national_registration_identity_card}
```

## 方法总结

面对 C++ 反编译噪声，应先识别 `std::cin`、`std::cout` 和 `std::string` 包装，再把注意力收窄到业务校验函数。编号校验题的识别信号包括固定前缀、数字位权、模 11 映射和末尾校验字母；构造一个最简单的通过样例能验证逆向公式，再据结构识别制度名称。
