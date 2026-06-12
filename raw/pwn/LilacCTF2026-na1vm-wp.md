# na1vm WP

## 题目简述

题目实现了一个自定义 VM，并提供提交指令、执行队列等交互。VM 支持 `store`、`load`、`pushadd`、`popadd`、`dumpreg`、`resetqueue` 等操作，任务队列元数据与 VM 寄存器/栈状态可以通过越界写互相影响。

核心漏洞是任务队列结构存在越界写。通过覆盖队列元数据，可以控制 VM 寄存器和栈基址，进一步用 VM 的 `push`/`pop` 类指令构造任意地址读写。

## 解题过程

首先封装 VM 指令。原官方脚本中 `pack_cmd(op, reg1, reg2, offset, imm)` 会把操作码、寄存器和立即数打包成 VM 命令，常用指令包括 `store`、`load`、`pushadd`、`popadd`、`dumpreg`、`resetqueue` 等。

利用过程分四步：

1. 通过队列越界写修改队列的 `head/tail/size`，让下一条命令写到队列结构外。
2. 使用 `dumpreg` 泄露 VM 栈或全局区指针，得到 ELF 基址。
3. 伪造读写基址后，用 `popadd` 从 ELF 中的标准 IO 指针附近泄露 libc 地址，并计算 libc base。
4. 因容器中 libc 与 ld 的映射相对稳定，可以由 libc base 推出 ld base，并利用默认注册在 `__exit_funcs` 中的 `dl_fini` 反推出 pointer mangling key。

拿到任意写和 mangling key 后，就可以劫持 `__exit_funcs`。官方脚本将要执行的命令先写入可控内存，再把 `__exit_funcs` 中的函数指针改成 mangled 后的 `system`，参数指向该命令字符串。最后通过 VM/IPC 协议调整状态触发 `exit()`，从而执行 `system("bash -c ...")` 或直接把 `/flag` 发回。

## 方法总结

本题的关键是先把 VM 的队列越界写升级成稳定任意读写，再处理 glibc 的退出函数指针保护。`__exit_funcs` 不是简单写入裸函数指针即可触发，需要用已有的 `dl_fini` 指针恢复 mangling key，再写入按同样规则编码后的目标地址。
