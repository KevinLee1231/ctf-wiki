# 开锁师傅

## 题目简述

附件是使用传统 ZipCrypto 加密的 ZIP，内部包含 PNG 文件 `huiliyi.png` 和 `flag.txt`。PNG 的文件签名与 `IHDR` 块头固定，可提供足够长的已知明文；`bkcrack` 能据此恢复 ZipCrypto 的三个内部 key，再直接解密同一压缩包中的 flag 条目。

## 解题过程

先构造 PNG 开头 16 字节的已知明文：

```bash
echo 89504E470D0A1A0A0000000D49484452 | xxd -r -ps > pngheader
```

其中前 8 字节是 PNG signature，后续为首个 `IHDR` 块的固定长度与类型。把明文文件与加密条目对应后执行：

```bash
bkcrack -C attachment.zip -c huiliyi.png -p pngheader
```

工具会从已知明密文关系恢复三个 ZipCrypto key，例如输出：

```text
Keys: cdc564be 5675041f 719adb56
```

使用恢复的 key 解密 `flag.txt`：

```bash
bkcrack -C attachment.zip \
  -k cdc564be 5675041f 719adb56 \
  -c flag.txt -d flag.txt
cat flag.txt
```

该攻击针对传统 ZipCrypto；如果压缩包使用 AES 加密，这组已知明文命令并不适用。

## 方法总结

- 核心技巧：利用文件格式固定头对传统 ZipCrypto 做已知明文攻击。
- 识别信号：加密 ZIP 内含 PNG、PDF 等已知格式文件，并使用 legacy ZipCrypto。
- 复用要点：已知明文必须与正确的压缩包条目和偏移对应；恢复的是压缩包内部 key，可继续解密同档案的其它条目。
