# GreyCTF2024 Short WP

## 题目简述

徽章提示把某个调试焊盘短接到地。固件读取 STM32 的 PB15 输入；当该引脚为低电平时立即输出 flag，并设置红色帽子解锁位。

## 解题过程

固件条件非常直接：

```c
if (HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_15) == 0) {
    usbPrintf(MAX_BUF_LEN, "grey{d3Bu9_P4D5}");
} else {
    usbPrintf(MAX_BUF_LEN, "Connect PB15 to GND");
}
```

断电时先确认 PCB 上 PB15 与 GND 的测试点，避免误接 3.3 V。用跳线连接 PB15 和 GND，再给徽章上电并从 USB 菜单进入 Challenge 2。输入被拉到逻辑 0，串口显示：

```text
grey{d3Bu9_P4D5}
```

读取结束后可移除跳线；这里不需要制造瞬态故障，稳定的低电平短接就是预期条件。

## 方法总结

遇到实物引脚题，先从固件确认端口、有效电平和内部上下拉，再动手接线。本题名虽有“Short”，目标是 PB15 对 GND 的受控短接，不应在未知焊盘之间试错，以免把电源轨直接短路。
