# ADS

## 题目简述

附件 `secret_message.rar` 解压后看似只有普通文件，真正的数据却藏在 NTFS Alternate Data Streams（备用数据流）中。三个数据流分别考查 ADS 提取、循环移位和音频频谱图，三段明文拼接后才是完整 flag。

## 解题过程

先在 NTFS 文件系统中用 PowerShell 列出数据流：

```powershell
Get-Item -LiteralPath ".\secret_message.txt" -Stream *
```

可见 `secret1.txt`、`secret2.txt`、`secret3.wav` 三个命名流。逐个读取或导出：

```powershell
Get-Content -LiteralPath ".\secret_message.txt" -Stream "secret1.txt"
Get-Content -LiteralPath ".\secret_message.txt" -Stream "secret2.txt"
Get-Content -LiteralPath ".\secret_message.txt" -Stream "secret3.wav" -AsByteStream -ReadCount 0 |
    Set-Content -LiteralPath ".\secret3.wav" -AsByteStream
```

第一个流给出首段及提示：

```text
grey{@lt3rnat3_dat@_5tr3ams
Hint for my next secret: Circular Shift
```

第二个流是：

```text
F5 25 F5 07 27 33 47 45 97 F5 36 03 F6 C4
```

把每个字节循环移位 4 位。由于半字节交换满足左旋 4 位等于右旋 4 位，解码式可写为：

$$
p=((c\ll4)\mathbin{\&}255)\mathbin{|}(c\gg4)
$$

```python
ciphertext = bytes.fromhex("F5 25 F5 07 27 33 47 45 97 F5 36 03 F6 C4")
plaintext = bytes(((value << 4) & 0xff) | (value >> 4) for value in ciphertext)
print(plaintext.decode())
```

输出中段 `_R_pr3tTy_c0oL`。最后把 `secret3.wav` 画成频谱图；图中没有需要保留的额外视觉结构，只显示一行清晰字符，转写为：

```text
_t0_H1d3_s3cret5!}
```

三段连接得到：

```text
grey{@lt3rnat3_dat@_5tr3ams_R_pr3tTy_c0oL_t0_H1d3_s3cret5!}
```

## 方法总结

- 核心技巧：保留 NTFS ADS 解压附件，枚举并导出命名流，再按各流提示分别解码。
- 识别信号：题名反复暗示 ADS，表面文件内容不足，而 NTFS 流列表存在额外名称；音频听感无意义但频谱呈清晰字符。
- 复用要点：普通复制、部分解压器或转移到不支持 ADS 的文件系统可能丢失命名流；应先验证流是否仍在，再做后续分析。
