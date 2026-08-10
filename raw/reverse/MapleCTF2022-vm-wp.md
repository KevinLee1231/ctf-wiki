# vm

## 题目简述

附件包含混淆命名的 SystemVerilog CPU、12 位程序 ROM `prog.txt` 和 8 位数据 RAM `data.txt`。本意是恢复一套栈式虚拟机并逆向 flag checker，但第一版在初始化数据中直接保留了明文 flag，形成非预期解。

## 解题过程

先看 `chal.sv` 的 `$readmemh` 调用，可以确认 `data.txt` 被直接载入 8 位 RAM。逐行把十六进制字节转为字符，在偏移 `0x8c`（十进制 140）处就能看到完整明文，不需要执行 VM：

```python
data = bytes(int(line, 16) for line in open("data.txt"))
print(data[0x8c:])
```

得到：

```text
maple{virtual_machine_more_like_verilog_machine}
```

从源码可以解释泄露原因：生成脚本原本把 flag 放入地址 140，checker 再从该地址读取并加密比较；发布附件时没有用加密后的数据替换这段初始 RAM。后续 `vm-v2` 只修改了 `data.txt` 来修复这个问题。

## 方法总结

面对 VM 题，先检查程序、数据和字符串等静态资源，再决定是否需要恢复完整指令集。初始化 RAM、测试向量、调试符号和生成脚本都可能造成直接泄露。本题的非预期解不代表 VM 分析无价值，但按最低成本证据顺序，静态扫描应当先于长时间的硬件仿真。
