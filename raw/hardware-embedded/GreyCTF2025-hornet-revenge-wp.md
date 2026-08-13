# Hornet Revenge

## 题目简述

题目在 RP2350 与 Lattice ECP5 FPGA 徽章上分四阶段执行：硬件知识问答、GPIO 短接、UART 交互和密钥提交。每一阶段返回 flag 的一部分。

## 解题过程

在 Thonny REPL 停止后台后导入题目模块：

```python
from hornet_revenge import *
```

运行 `qna1()`，依次回答：

```text
RP2350
programmable
field programmable gate array
verilog
lfe5u-25f-6bg256c
```

得到第一段 `grey{for_last_greyctf_`。运行 `qna2()` 后按提示把 RP2350 的 `GP27` 短接到 GND，得到第二段 `i_was_`。

`qna3()` 要求从 FPGA 取密钥。用 GP8/GP9 以 9600 波特初始化 UART，并发送手册中的 Read Data 指令，选择器 `A` 表示读取 Hornet Revenge 密钥：

```python
import busio, board
uart = busio.UART(board.GP8, board.GP9, baudrate=9600, timeout=0.1)
uart.write(b"@---------------A@")
print(uart.read())
```

读取到：

```text
{hi_i'm_your_army}
```

最后运行 `qna4()` 并提交该密钥，得到末段 `holding_back...but_this_greyctf_i'm_no_longer_sleep_deprived}`。拼接为：

```text
grey{for_last_greyctf_i_was_holding_back...but_this_greyctf_i'm_no_longer_sleep_deprived}
```

## 方法总结

这题的重点是按阶段维护物理状态：先确认 MCU、PIO 与 FPGA 型号，再完成指定 GPIO 拉低，最后按正确引脚方向和波特率建立 UART。UART 的第一个参数是 TX、第二个是 RX；指令必须包含完整的 `@...A@` 18 字符结构。各段 flag 应按返回顺序原样拼接，不能擅自补删下划线或标点。
