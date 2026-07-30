# NepCTF2026 UnknownFirmware Writeup

## 题目简述

附件是一份缺少常见文件头的嵌入式固件。字符串和入口附近的特征表明它采用 Telink `KNLT` 格式，处理器指令集为 TC32。目标是恢复命令处理函数中的校验逻辑，得到 AES 密钥，再解密固件内嵌资源中的二维码。

## 解题过程

先从固件中的 `KNLT` 标记、向量表布局和代码特征识别出 Telink 平台。GNU binutils 可强制按 TC32/Thumb 风格反汇编：

```bash
tc32-elf-objdump -D -b binary -m tc32 -M force-thumb \
  --start-address=0x360 firmware.bin
```

固件大小为 `0x4450`，主循环会复制一个 `0x40` 字节的射频包交给 `0x26a0`。该函数要求 `packet[0] == 0x10`，真正的 16 字节候选密钥位于 `packet[4:20]`；其余包字节不参与密钥检查。

校验不是直接比较密钥。它以 `0x5b25` 初始化非反射 CRC-16/CCITT，并对 16 个位置依次生成下一个素数：

```text
41, 43, 47, 53, 59, 61, 67, 71,
73, 79, 83, 89, 97, 101, 103, 107
```

设当前素数为 $p_i$、密钥字节为 $k_i$，送入 CRC 的字节为：

$$
t_i=p_i\oplus((p_i+k_i)\bmod256).
$$

固件在每加入一个 $t_i$ 后，就把当前 CRC 与 `0x3448` 处的前缀表比较。已知上一步 CRC 和本次目标值时，只需枚举 256 个 $t_i$ 候选。恢复出的中间字节为：

```text
63 52 bb 90 92 9f f0 d9 e3 f2 94 95 db b1 bb e7
```

再逐字节求逆：

$$
k_i=((p_i\oplus t_i)-p_i)\bmod256,
$$

得到 AES-128 密钥：

```text
!NepnepWantsYou!
```

对应的合法触发包结构为：

```text
10 00 00 00 || "!NepnepWantsYou!" || 44 字节 00
```

这 16 字节同时被送入硬件 AES 外设。对 `0x800540`、`0x800548`、`0x800550..0x80055f` 的访问顺序进行交叉检查，可确认代码设置解密模式、写入数据 FIFO 和 16 字节 key。使用标准 AES-128-ECB 解密固件 `0x3470` 起的 `0x3d0` 字节资源：

```python
from Crypto.Cipher import AES

key = b"!NepnepWantsYou!"
ciphertext = firmware[0x3470:0x3840]
payload = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
```

明文共 976 字节，并非普通灰度图。固件绘图函数给出的宽、高均为 90，但每行只取 `90 >> 3 = 11` 字节，共需 990 字节，因此先在尾部补 14 个零。随后：

1. 把每个字节按 `b7b6 | b5b4 | b3b2 | b1b0` 分成 4 对；每对均为 `00` 或 `11`，折叠成一个逻辑 bit；
2. 将 3960 个 bit 重排为 $45\times88$；
3. 每两行完全重复，折叠为 $45\times44$ 个二维码模块；
4. 反转电子纸黑白极性并补静区。

最终得到二维码：

![折叠重复像素和重复行、反色并补静区后恢复出的 flag 二维码](./NepCTF2026-unknown-firmware-wp/decoded-qr.png)

扫码结果为：

```text
NepCTF{U_Have_Dec0mpiled_Telink_FirmWareWooooW}
```

## 方法总结

本题的决定性步骤是正确识别无文件头固件的架构和厂商格式。完成架构识别后，射频包处理、前缀 CRC、连续素数变换和 AES 外设访问能够相互印证。密钥保护只有 `16 × 256` 次单字节枚举强度；真正容易误判的是电子纸的重复 bit、重复行与反色布局。二维码具有空间排列信息，归档时保留图片有意义，而反汇编和终端输出均可直接转写为文本。
