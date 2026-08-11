# ESPecially secure boot

## 题目简述

题目模拟启用 Secure Boot V1、未启用 Flash Encryption 的 ESP32 启动链。上传内容会被写入 flash 的应用镜像区，再由 ESP-IDF v3.1-rc2 的二级 bootloader 装载。该旧版本在验证镜像签名之前没有充分约束每个 image segment 的 load address：攻击者可让一个经过格式化的 segment 覆写 bootloader 已加载的签名验证代码，从而令未签名的应用执行并从 flash 读取 flag。

决定性障碍是 ESP32 镜像装载和 Secure Boot 信任边界，而不是通用 Linux 进程利用，故按 hardware-embedded 归类。

## 解题过程

### 镜像结构与漏洞位置

官方脚本首先读取合法的 `payload-unsigned.bin`：24 字节头部以 `0xe9` 开始，随后每段为小端 `load_addr`、`data_len` 和段数据。它保留原段、把 header 的 segment count 加一，并额外加入一个可控段。bootloader 先根据段头把数据拷入目标加载地址，之后才进行 Secure Boot 校验；因此伪造段的写入早于被覆盖的验证调用。

### 绕过签名校验并执行应用

额外段选择加载地址 `0x4007b7ca`、长度 4，并写入与该 Xtensa 指令位置匹配的短跳过/返回补丁。官方脚本随后按 ESP image 规则重新计算：对所有 segment 数据异或得到 checksum byte，补齐到 16 字节边界，附加 checksum 与 SHA-256，并在尾部填充伪造的 64 字节签名域。

上传的应用 `app_main` 只做两件事：打印 secure boot 已被绕过的提示，并以 `spi_flash_read(0x133370, flag, 0x100)` 读取 flag。脚本反复向服务发送 Base64 镜像，是为了适配 QEMU 执行中的非确定性启动状态；它以输出中 `epc1=` 的异常迹象判定一次尝试失败，而非声称每次都稳定命中。

### 验证

题目配置给出的 flag 为 `DUCTF{can_you_exploit_without_the_-seed_arg_set?}`。本文没有启动 QEMU、写 flash 或执行上传的 firmware；内容仅由服务包装脚本、官方镜像构造脚本和应用源码静态交叉确认。

## 方法总结

- 核心技巧：Secure Boot 只保护“被正确验证的映像”；若装载器先按攻击者给出的地址写内存，签名检查本身也可被加载过程反向破坏。
- 识别信号：老版本 bootloader、Secure Boot 已开而 Flash Encryption 关闭、可提交原始 image segment 时，应优先检查 segment load address 的范围验证与验证顺序。
- 复用要点：ESP image 的段头、checksum、16 字节 padding 和哈希域必须保持一致；覆盖地址与指令字严格绑定特定 ESP-IDF、ROM 和 QEMU 版本。
