# GreyCTF2024 Bootloader WP

## 题目简述

正常菜单要求先输入 flag 才允许重启进入 bootloader，形成“需要 flag 才能读到含 flag 的固件”的循环。徽章使用 DAPBoot，配置为上电时检测 STM32F103 的 BOOT1/PB2；把该引脚拉高可以绕过应用层口令，直接进入 USB DFU 启动器。

## 解题过程

先断电，把 BOOT1 接到 3.3 V 后复位徽章。公开配置中启用了 `HAVE_BUTTON`、`BUTTON_ACTIVE_HIGH 1`，检测端口为 GPIOB、引脚为 GPIO2，因此高电平会强制停留在 DAPBoot。用 DFU 主机工具确认设备并上传可读的应用固件：

```bash
dfu-util --list
dfu-util -a 0 -U firmware.bin
```

不同系统上 DFU alternate setting 的编号可能不同，应以 `--list` 输出为准。取得固件后搜索 30 字节比较循环，会看到输入逐字节满足：

```c
input_ans[i] ^ boot_flag_array1[i] == boot_flag_array2[i]
```

因此明文字节为两组数组异或：

```python
a = [20,99,10,94,20,9,46,98,9,7,57,56,30,100,86,88,48,41,56,72,31,78,26,9,4,86,14,54,100,34]
b = [115,17,111,39,111,126,70,3,125,88,95,84,127,3,9,50,69,90,76,23,125,33,117,125,53,9,108,68,11,95]
print(bytes(x ^ y for x, y in zip(a, b)).decode())
```

输出为：

```text
grey{what_flag_just_boot1_bro}
```

## 方法总结

应用菜单的访问控制不会自动保护启动链。只要硬件启动条件仍允许进入可上传固件的 DFU bootloader，就能绕过应用层口令。分析时应沿“启动引脚—bootloader 配置—DFU 权限—固件比较逻辑”建立完整链路，而不是只盯着菜单校验。
