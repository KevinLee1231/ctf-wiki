# rd

## 题目简述

题目是一个以空行分隔、字段格式为 `key:value` 的多用户任务服务，支持 `register`、`login`、`deregister` 和 `submit_task`。任务由后台线程异步处理，内容在写入结果前会按字节异或 `0x3f`。

程序同时存在三类生命周期错误：注销只释放 user 结构体而不清理其成员；全局 task 数达到上限后，`allocate_task` 直接返回却不把新 user 的 `tasks` 置空；后台 `do_task/task_log` 又会在较长时间后继续使用 task。官方预期利用共享 task 的竞态和 UAF，总 PDF 则用堆布局直接控制未初始化的 `tasks` 指针，分两轮伪造 `stdout` 完成泄露与 House of Apple 2。

## 解题过程

### 1. 还原未初始化 task 指针

反编译得到的分配逻辑为：

```c
void allocate_task(user_t *user) {
    task_t *task;
    if (task_allocated <= 15) {
        task = malloc(0x28);
        task->result = 0;
        task->task_length = 0;
        task->user_running = &user->task_running;
        ++task_allocated;
        user->tasks = task;
    }
}
```

前 16 次会初始化 `user->tasks`；从第 17 个 user 起，函数什么也不做。如果该 user 结构体复用了先前释放或由协议解析器填充的同尺寸堆块，`tasks` 就保留旧内容，后续提交/运行任务会把它当作真实 `task_t *`。

注销路径只 `free(user)`，没有释放或断开已有 task；因此官方路线可让新旧 user 共享 task。后台线程在 `task_log` 中停留较久，主线程可在它睡眠期间注销、重分配并改写原对象，形成异步 UAF 和堆溢出。

### 2. 官方预期：竞态建立堆读写

题目目录的单题 WP 按三阶段利用竞态：

1. 注册 16 个用户耗尽正常 task 分配；取得若干 token，并提交不同大小的异步任务。
2. 在任务仍运行时注销其 user，再注册同尺寸新 user 复用堆块。后台线程醒来后把异或还原的长结果写进已被复用的对象，覆盖相邻 user 的 token 或指针。
3. 第一轮覆盖令登录响应泄露 safe-linking 值，左移 12 得到 heap base；第二轮把 victim 指针改到堆内 unsorted-bin 元数据，泄露 main_arena 并还原 libc；第三轮伪造 tcache next，把分配目标导向 `_IO_2_1_stdout_`，再布置 House of Apple 2。

官方脚本在每轮提交后等待约 4.5 秒，以覆盖后台任务的延迟窗口。这个等待值不是漏洞条件，应以服务实际任务时长为准；更稳的实现应根据可观察响应推进阶段，无法观测时再使用有上下界的重试。

### 3. 总 PDF：不打 race 的堆复用路线

总 PDF 先注册 16 个用户，再用三个用户提交约 `0xa00`、`0xa10`、`0xa20` 字节任务，制造大块和异步 worker 布局。随后发送缺少合法 `command` 的畸形字段包，让协议解析路径分别释放或保留特定大小的临时块。

第 17 个 user 不再获得新 task。通过上述堆风水，让它复用攻击者填充过的 user 块，就可把残留 `tasks` 设置为 `stdout - 0x20`。后续 `submit_task` 对“task 结果”的写入实际落入 libc 的标准流对象附近。虽然没有利用两个线程同时改同一 task，后台任务仍是异步的，所以远程脚本仍需在关键提交后等待，成功率也会受堆分配与调度影响。

### 4. 先泄露 libc，再伪造 stdout 泄露 heap

登录 token 的输出长度处理不严，堆布局合适时会把 token 后方的 unsorted-bin 指针一并打印。PDF exploit 从 `token[32:]` 取泄露，并使用附件 glibc 2.39 的偏移：

```text
libc_base = leak - 0x203b20
stdout    = libc_base + 0x2046a8
```

将未初始化 `tasks` 指到 `stdout - 0x20` 后，提交的 task 内容会异或 `0x3f`，所以发送前必须把目标 `_IO_FILE` 每字节再异或一次。第一份 fake stdout 设置：

- `_flags = 0xfbad1802`，文件描述符为 1；
- `_IO_write_base` 指向 libc 中保存 heap 指针的全局位置；
- `_IO_write_ptr = _IO_write_end = base + 8`；
- `_lock` 和 vtable 指向该 libc 版本中的合法可读写对象。

下一次正常登录触发 stdout 输出时，会把 `[write_base, write_ptr)` 的 8 字节当作待输出数据，得到 heap address。这样第一轮 FILE 伪造只负责读，不承担控制流劫持，便于检查泄露是否正确。

### 5. 第二轮 House of Apple 2

取得 heap 后，重复一次堆复用，把另一名未初始化 user 的 `tasks` 再次指向 `stdout - 0x20`。在已知堆地址布置 fake `_IO_FILE`、fake `_IO_wide_data` 和 fake wide vtable：

- FILE 开头放可被 `system` 解释的命令字符串，PDF 使用等价于 `"  sh;"` 的 `_flags` 字节；
- `_IO_write_ptr > _IO_write_base`，使下一次输出进入 overflow 路径；
- `_wide_data` 指向堆上的伪造 wide-data；
- vtable 使用合法的 `_IO_wfile_jumps`，绕过 vtable 合法性检查；
- fake wide vtable 的调用槽填入 `system`。

附件环境对应的主要偏移为：

```text
system          = libc_base + 0x58750
_IO_wfile_jumps = libc_base + 0x202228
```

触发一次正常输出后，wide FILE 路径最终以 fake FILE 地址为参数调用 `system`，FILE 起始字节即 shell 命令，由此获得 shell。总 PDF 最后一页的终端截图显示成功执行 `cat flag.txt`，flag 为：

```text
RCTF{y0u_4re_master_0f_r4ce_condition}
```

### 6. 稳定性与调试边界

PDF 脚本区分本地和远端的 fake FILE 堆偏移，例如 `heap_base + 0x6240` 与 `heap_base + 0x5e30`，说明其布局并不通用。稳定复现应做到：

1. 每个阶段独立验证泄露是否落在 heap/libc 合法区间；失败立即重连，不带错误基址继续写。
2. 把“耗尽 task”“制造 unsorted chunk”“控制第 17 个 user”“第一轮 stdout 泄露”“第二轮 stdout 劫持”分成可重试阶段。
3. 使用随题 glibc 2.39 重新确认 `_IO_FILE`、`_IO_wide_data`、`_IO_wfile_jumps` 和 `system` 偏移。
4. task 内容落地前会异或 `0x3f`，所有 fake FILE 字节必须预编码；协议字段本身不应误做相同变换。

## 方法总结

- 根因是 user、task 与后台 worker 的生命周期不一致：成员未清理、配额失败未初始化、异步线程继续使用旧对象。
- 官方预期利用竞态/UAF 逐步得到 heap、libc 和 tcache poisoning；总 PDF 通过堆复用直接控制未初始化 task 指针，省去显式竞态但仍受异步时序影响。
- 两轮 FILE 伪造职责不同：第一轮把 stdout 变成定长任意读以泄露 heap，第二轮才用 House of Apple 2 调用 `system`。
- 远程低成功率不是“多跑几次即可”的结论；应检查每次泄露、对象地址和 FILE 字段，再决定重试，避免把错误状态推进到不可恢复阶段。
