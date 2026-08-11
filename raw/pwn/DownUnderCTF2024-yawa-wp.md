# yawa

## 题目简述

菜单服务在选项 1 对 `char name[88]` 调用 `read(0, name, 0x88)`；长度 `0x88=136` 已越过数组，也不会写 `\0`。选项 2 随即用 `printf("Hello, %s\\n", name)` 输出。因此第一次只覆写 canary 的低零字节以泄露余下字节，第二次泄露返回地址以计算 libc 基址，最后以保留 canary 的栈溢出构造 ret2libc。

随附 libc 和官方 solver 给出了本题构建所用偏移；ASLR、canary 与栈对齐是利用链的必要部分。

## 解题过程

### 两次非终止字符串泄露

第一次选择 1 并发送 89 个字节。89 字节恰好越过 88 字节数组一个位置，将 canary 的固定低位 `\0` 替换成非零字节，却不改动剩余 7 字节；当 `printf("%s")` 继续打印时，官方 solver 从输出尾部的 7 个原始非零字节复原 canary，再左移 8 位补回固定低位零字节：

```python
canary = u64(leak[-9:-2].ljust(8, b"\0")) << 8
```

第二轮以 104 个字节覆盖到保存的返回地址区域，并再次使用选项 2 读取尾部 6 字节。solver 以该泄露减去 `0x29d90` 得到随附 libc 的加载基址。

### 带 canary 的 ret2libc

源码中 `name` 到 canary 的距离为 88 字节。最终名称缓冲区写入：88 字节填充、泄露的 canary、保存帧填充、单个 `ret` 用于 ABI 栈对齐、`pop rdi; ret`、`/bin/sh` 地址、`system` 地址。官方 solver 针对给定 libc 使用：

```python
ret     = libc_base + 0x1bc065
pop_rdi = libc_base + 0x1bbea1
bin_sh  = next(libc.search(b"/bin/sh"))
system  = libc.symbols["system"]
```

提交最终 payload 后选择非 1、2 的菜单项离开循环，函数返回并进入 ROP 链。

### 验证

官方脚本将交互切至取得的 shell；题目配置中的 flag 为 `DUCTF{Hello,AAAAAAAAAAAAAAAAAAAAAAAAA}`。本文没有连接服务或执行 ROP，以上偏移和结果均来自源码、随附 libc 与官方 solver 的静态对照。

## 方法总结

- 核心技巧：超过栈数组的 `read` 与 `%s` 的组合可把“覆盖掉 canary 低位零字节”转化为信息泄露；先泄露 canary 与 libc，再进行受保护的溢出。
- 识别信号：固定栈数组、`read` 精确写满数组、随后 `%s` 打印，是典型的越界读加 ret2libc 链。
- 复用要点：泄露截取位置、返回地址减数和 gadget 偏移都绑定本题的 libc 与编译布局；`ret` 对齐 gadget 不能随意省略。
