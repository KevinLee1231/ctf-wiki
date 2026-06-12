# Aberration

## 题目简述

题目给出 `qemu_fw.bios` 固件、ARM64 kernel Image 和 rootfs，需要从固件镜像中提取 BL31，对 ARM Trusted Firmware 的 SMC handler 做逆向。附件 `run.sh` 以 `qemu-system-aarch64` 启动 4 核虚拟机，参数包含 `-machine virt,secure=on,gic-version=3` 和 `-cpu max,sme=on,pauth-impdef=on`，说明题目目标在 TrustZone/EL3 侧而不是普通 Linux 用户态。

题目 README 明确说明 flag 藏在系统寄存器 `S3_3_C15_C12_X`，只有 EL3 代码能通过 `mrs` 指令读取。漏洞位于 `aberration_smc_handler` 调用链中的 `ark_store`：长度检查和后续 `memcpy` 之间存在竞态窗口，可跨 CPU 修改传入的 `key_len_ptr`，最终破坏 `g_aberration_area.buffer` 附近的管理字段。

## 解题过程

### 关键观察

1. 从 `qemu_fw.bios` 中提取 BL31 后，定位 `aberration_smc_handler` 处理 SMC 请求的逻辑。运行环境是 4 核 ARM64 secure machine，因此跨 CPU 竞态是可触发条件。
2. `ark_store` 在 `memcpy` 前会检查长度，但实际拷贝前仍会从用户可影响的位置读取长度或相关指针，形成 TOCTOU。
3. 利用需要跨 CPU 触发：一个 CPU 反复把 `key_len_ptr` 改为异常值，另一个 CPU 反复调用 `ark_store`。如果修改落在检查之后、拷贝之前，异常长度会写入 `g_aberration_area.buffer`，进一步污染 `max_slots`。

### 利用链

污染 `max_slots` 后，`ark_add` 的槽位边界检查会失效，由此获得任意地址写能力。后续利用不依赖常规用户态 ROP，而是伪造页表项，把可执行代码段重新映射为可读、可写、不可执行位被调整后的权限组合，再写入 shellcode。

最终 shellcode 在 EL3 执行 `mrs` 读取系统寄存器 `S3_3_C15_C12_X` 取得 flag 值，再把结果复制到普通世界可读的内存区域。当前可复现的公开主线就是：固件提取 BL31、SMC handler 竞态、破坏 `max_slots`、借 `ark_add` 任意写、页表伪造和 shellcode 读寄存器。

## 方法总结

- 核心技巧：ARM 固件 SMC handler 中的跨 CPU TOCTOU，利用管理字段损坏扩大为任意写，再通过页表权限伪造执行 shellcode。
- 识别信号：固件题出现 BL31、SMC handler、共享缓冲区和长度指针时，应检查检查点和使用点之间是否能由另一 CPU 改写。
- 复用要点：竞态利用需要明确两个线程/CPU 的角色；获得任意写后，固件环境常见的后续路线是改页表权限或映射关系，而不是寻找用户态返回链。
