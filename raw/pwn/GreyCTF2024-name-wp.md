# GreyCTF2024 Name WP

## 题目简述

徽章允许用户设置显示名称，但把用户输入直接当作格式字符串传给 `usbPrintf`。同一次调用又把解密后的 flag 指针作为第一个可变参数，因此输入 `%s` 就能把该指针指向的字符串打印出来。

## 解题过程

固件先异或两组数组生成 flag，然后执行：

```c
usbPrintf(MAX_BUF_LEN, "\nEnter your name> ");
readString(input_ans, STATE_NAME_SIZE - 1);
usbPrintf(MAX_BUF_LEN, input_ans, flag);
```

安全写法应是 `usbPrintf(..., "%s", input_ans)`；当前代码却让 `input_ans` 控制格式。输入：

```text
%s
```

格式化函数会从第一个可变参数取一个字符指针，而该参数正好是 `flag`，于是 USB 串口输出：

```text
grey{you_can_printf_on_stm32?}
```

固件还显式检查输入中是否包含 `%s`，命中后设置蓝色帽子解锁位，说明这正是预期解法。

## 方法总结

格式化字符串漏洞的关键不只是“用户控制格式”，还要分析后续变参栈中有什么。本题已经把 flag 指针作为第一个参数传入，所以无需遍历 `%p` 或猜偏移，一个 `%s` 即可完成定向信息泄露。
