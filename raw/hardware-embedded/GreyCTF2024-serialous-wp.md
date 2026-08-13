# GreyCTF2024 Serialous WP

## 题目简述

本题需要两块徽章：一块通过 USART1 发送秘密消息，另一块作为串口接收端。关键是正确交叉 TX/RX 并共地，然后读取发送端固件在运行时异或生成的字符串。

## 解题过程

接线方式为：

```text
发送徽章 TX  -> 接收徽章 RX
发送徽章 GND -> 接收徽章 GND
```

接收徽章进入 USART 读取工具，发送徽章选择 `Send Serialous Message`。固件执行：

```c
for (int i = 0; i < UART_FLAG_LEN; i++)
    data[i] = uart_flag_arr1[i] ^ uart_flag_arr2[i];
HAL_UART_Transmit(&huart1, data, UART_FLAG_LEN, 200);
```

接收端得到：

```text
grey{uart_read}
```

也可用公开固件中的两组 15 元素数组离线异或验证这一结果。挑战目录的 README 把 Flag 字段误写成了 Challenge 1 的 `grey{STM32_ExP3R1}`；实际 UART 数据与源码注释都明确表明本题答案是 `grey{uart_read}`。

## 方法总结

UART 连接至少需要交叉数据线和公共参考地；只连 TX/RX 而不共地可能造成不稳定电平。赛后资料发生矛盾时，应以实际数据流和生成代码为主，而不是机械复制 README 的元数据字段。
