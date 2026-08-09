# Strings

## 题目简述

主函数直接执行 `printf(buf)`，形成格式化字符串漏洞。随后 `main2` 把真实 flag 读到栈上，却以可写的全局字符串 `fake_flag` 作为格式串再次调用 `printf`。

## 解题过程

第一次格式化字符串的参数偏移为 6。利用 `%n` 写原语，把全局 `fake_flag` 从假 flag 改写为 `%s`：

```python
payload = fmtstr_payload(6, {elf.symbols.fake_flag: b"%s"})
io.sendline(payload)
```

进入 `main2` 后，程序先执行 `fgets(flag, 40, flag_ptr)`，真实 flag 位于栈上；随后 `printf(fake_flag)` 实际变成 `printf("%s")`。这本来是缺少可变参数的未定义行为，但题目给定二进制的调用现场中会取到指向栈上 flag 的残留指针，从而泄露：

```text
n00bz{f0rm4t_5tr1ng5_4r3_th3_b3s7!!!!!}
```

## 方法总结

格式化字符串不仅能读栈，还能用 `%n` 改写后续控制数据。本题利用依赖具体编译结果和寄存器残留，因此复现应以随题二进制为准，不能假设这段未定义行为在任意构建中都稳定。
