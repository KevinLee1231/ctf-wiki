# Hidden in Plain Sight

## 题目简述

题目使用 GreyFlag 徽章上的 ESP32-C6，把 flag 的十六进制主体写入芯片 eFuse。eFuse 是片上一次性可编程配置存储，与普通 SPI flash 分离，因此重刷题目固件不会清除该值。题面要求把读出的数封装为 `grey{0x<value>}`。

## 解题过程

用 USB 连接徽章，先确认系统分配的串口，再调用 Espressif 的 `espefuse` 工具直接读取整份 eFuse 摘要：

```bash
espefuse --chip esp32c6 --port <serial-port> dump
```

命令会显示各 eFuse 区域的原始十六进制内容。题目没有再叠加加密或字节序变换，异常且可打印为题目答案的连续值为：

```text
C0FFEE414141676767DEAD
```

按题面保留 `0x` 前缀并包入 flag 格式：

```text
grey{0xC0FFEE414141676767DEAD}
```

整个过程只读，不需要执行 `burn_efuse`、写 flash 或修改任何熔丝。若串口无法打开，应先关闭占用该端口的串口监视器，再重新执行 dump。

## 方法总结

这题考查的是存储边界识别：固件镜像、NVS 和 eFuse 是不同层次的数据源。看到“重刷后仍存在”的设备秘密时，应优先检查芯片的 OTP/eFuse 区域。`dump` 已经给出原始值，后续只需严格遵循题面格式；不应在真实设备上尝试任何不可逆的 eFuse 写操作。
