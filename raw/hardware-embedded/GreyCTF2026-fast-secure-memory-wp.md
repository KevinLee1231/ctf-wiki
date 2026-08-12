# Fast Secure Memory

## 题目简述

FPGA 内有一个同步 ROM，5 位地址选择一个字节。设计者试图只允许在地址 31 时看到内部输出，其余地址均返回 `?`：

```verilog
always @(posedge clk)
    value <= memory[address];

assign output = (address == 5'd31) ? value : "?";
```

问题在于 `value` 是时钟沿更新的寄存器，而访问检查直接使用当前组合地址。数据对应“上一个被时钟采样的地址”，权限判断却对应“此刻地址”，两者之间存在 TOCTOU 时序窗口。

## 解题过程

若把地址长期停在 $i$，ROM 会在 FPGA 的 50 MHz 上升沿把 `memory[i]` 装入 `value`，但输出门控仍因地址不是 31 而显示 `?`。此后迅速把地址改为 31，在下一次上升沿覆盖 `value` 之前采样输出，就会满足：

```text
寄存器 value = memory[i]
当前 address = 31
```

于是门控放行了本应属于地址 $i$ 的字节。CircuitPython 普通 GPIO 和 `sleep` 难以稳定命中几十纳秒级窗口，官方解法用 RP 系列 PIO 固定执行节拍：

```text
.program fast_secure_memory_probe
loop:
    pull block
    set pins TARGET_ADDRESS
    nop [7]
    set pins 31
    in pins, 7
    push block
    jmp loop
```

PIO 以 25 MHz 运行，地址线从 `GP8` 开始连续占 5 位；采样窗口从 `GP22` 至 `GP28` 读取 7 位。PMOD J2 到 GPIO 的布线不是顺序映射，需按官方脚本重排：

```python
value  = ((raw >> 5) & 1) << 0
value |= ((raw >> 2) & 1) << 1
value |= ((raw >> 1) & 1) << 2
value |= ((raw >> 3) & 1) << 3
value |= ((raw >> 4) & 1) << 4
value |= ((raw >> 6) & 1) << 5
value |= ((raw >> 0) & 1) << 6
```

当前 bitstream 还把原本未接出的 ASCII bit 1 镜像到已连接输出，因此 7 位已经足以还原可打印字符。对地址 0 至 14 各重复采样约 200 次，丢弃稳定的 `0`、`?` 和地址 31 自身值造成的干扰，对剩余样本做直方图并取众数：

```python
for addr in range(15):
    hist = Counter(raw_to_byte(x) for x in sample_raw(addr, 200))
    for noise in (0, ord("?"), ord("L")):
        hist.pop(noise, None)
    recovered.append(chr(hist.most_common(1)[0][0]))
```

按地址顺序拼接得到：

```text
grey{fasttimin}
```

## 方法总结

访问控制必须与被保护数据使用同一拍、同一地址语义。这里同步读取把数据延迟了一拍，组合门控却没有同步保存相应地址，使攻击者能先让敏感字节进入寄存器，再用“合法地址 31”通过检查。PIO 的作用不是破解逻辑，而是把地址切换与采样压缩到稳定、可重复的硬件时序；多次采样和众数统计则吸收了两个时钟域相位不固定带来的噪声。
