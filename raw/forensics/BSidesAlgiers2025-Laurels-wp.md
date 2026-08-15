# BSidesAlgiers2025 - Laurels

## 题目简述

题目给出被破坏的 `pcapng`，场景是攻击者向 Laureate Panel 显示器写入文字。必须先修复抓包，再把非标准 TCP 端口按 Modbus/TCP 解码，最后从第一块面板的显示寄存器中按时间顺序读出字符。

题面要求将恢复文字包装为 `shellmates{...}`。

## 解题过程

### 修复抓包

解开 `laurels.7z` 后，用 `pcapfix` 修复块长度、未知块、缺失的 Section Header 与 Interface Description Block：

```bash
pcapfix -n laurels.pcapng
```

本地复核结果显示共修复 6 处损坏，输出 `fixed_laurels.pcapng`，其中包含 20,966 个以太网帧。原始文件不能直接用于后续过滤，因此这一步不是可省略的格式清理。

### 恢复 Modbus 语义

同一设备地址使用三个非标准端口：

- TCP `504` 对应 Unit ID `2`；
- TCP `505` 对应 Unit ID `0`，即题面所说的第一块面板；
- TCP `506` 对应 Unit ID `1`。

需要在 Wireshark 中把这三个端口“Decode As” Modbus/TCP。厂商 [Modbus 手册](https://www.laurels.com/downloadfiles/Modbus-Manual-CTR.pdf) 第 25 页的写寄存器表明确列出：寄存器 `106`（`0x006a`）和 `107`（`0x006b`）分别承载显示数据的高、低字。正文已给出解题所需含义，链接只用于核对原始设备文档；另一个已不可访问的厂商网页不再保留。

官方显示过滤器为：

```text
modbus.func_code == 16 && modbus.reference_num == 106 && tcp.dstport == 505
```

若命令行版 tshark 因非标准端口无法正确区分查询与响应，可以直接按 Modbus/TCP PDU 的固定偏移筛选。MBAP 头占 7 字节，第 8 字节是功能码，第 9～10 字节是起始寄存器：

```bash
tshark -r fixed_laurels.pcapng \
  -Y "tcp.dstport == 505 && tcp.payload[7] == 0x10 && tcp.payload[8:2] == 00:6a" \
  -T fields -e tcp.payload
```

本地实测命中 21 条写请求。每条请求只写一个寄存器，最后一个字节依次为：

```text
S3cuR1tYisTh3K3y$t0n3
```

按题面包装后的最终 flag 为：

`shellmates{S3cuR1tYisTh3K3y$t0n3}`

## 方法总结

- 取证载体损坏时，先修复容器结构并记录修复数量，再讨论协议内容；跳过修复会把解析失败误判为“没有目标流量”。
- 非标准端口不会自动获得正确的应用层语义，应主动做 Decode As，并用 Unit ID、功能码和寄存器号共同收窄范围。
- 厂商文档在本题中的作用是把寄存器号映射为真实业务含义；一旦确认 `106/107` 是显示数据，剩余工作就是按帧顺序提取字符。
