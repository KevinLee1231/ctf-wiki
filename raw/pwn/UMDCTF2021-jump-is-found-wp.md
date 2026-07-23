# Jump Is Found

## 题目简述

题目先在堆上分配相邻缓冲区，前一个缓冲区可溢出覆盖后一个；随后数据被复制到栈并直接作为 `printf` 的格式字符串。利用需要先泄漏 libc，再借 `%hn` 对 `exit@GOT` 做分段写入。

## 解题过程

分析对象布局可见，第一次输入能够越过堆块边界，把后续将传给 `printf` 的内容替换成攻击者字符串。通过枚举位置参数找到 libc 泄漏位于：

```text
%51$p
```

用泄漏减去对应符号/返回地址偏移得到 libc 基址，并选择满足寄存器与栈约束的 one-gadget。

目标字符串还会经过 `strcpy`，地址中的 NUL 字节限制了直接一次写满 8 字节。将 one-gadget 地址拆成三个 16 位半字，按写入值升序排列，并为每个目标地址使用 `%hn`：

```python
parts = [
    (one_gadget & 0xffff, exit_got),
    ((one_gadget >> 16) & 0xffff, exit_got + 2),
    ((one_gadget >> 32) & 0xffff, exit_got + 4),
]
parts.sort()
```

格式化输出累计字符数依次达到各半字值，覆盖完成后让程序调用 `exit`，控制流转到 one-gadget。取得 shell后读取：

```text
UMDCTF-{1_f0UnD_th3_PLaN3t_N0w_t0_hyp325p4c3}
```

## 方法总结

这题把堆溢出和格式化字符串串成同一利用链。堆溢出负责控制格式字符串，`%p` 负责泄漏，`%hn` 负责绕过 NUL 和大数打印限制。分段写必须同时处理参数偏移、累计字符数回绕和写入顺序。
