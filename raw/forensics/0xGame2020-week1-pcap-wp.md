# week1pcap

## 题目简述

附件是包含 47552 帧的 Ethernet PCAP，其中有大量针对 `/upload/upload.php` 的盲注 GET 请求作为干扰。真正有价值的是第 20457 帧的一个 `multipart/form-data` POST：客户端上传了 `flag.zip`，压缩包内含被篡改 IHDR 高度的 `flag.png`。

## 解题过程

先在 Wireshark 使用显示过滤器缩小范围：

```text
http.request.method == "POST"
```

可以定位到第 20457 帧，请求路径为 `/upload/upload.php`，multipart 字段中的文件名为 `flag.zip`，类型为 `application/zip`。大量形如 `id=1' and ascii(substr(...))=...` 的 GET 请求与恢复图片无关。

在 Wireshark 中选择“文件 → 导出对象 → HTTP”，保存 `flag.zip`；也可以对第 20457 帧执行“追踪 TCP 流”，从 multipart 请求体中提取以 `PK\x03\x04` 开头的 ZIP 数据。解压后得到 `flag.png`。

PNG 签名后的第一个块是 IHDR。其宽、高字段分别位于文件偏移 `0x10` 和 `0x14`，都是 4 字节大端整数。题目图片的头部为：

```text
89 50 4E 47 0D 0A 1A 0A
00 00 00 0D 49 48 44 52
00 00 02 80 00 00 02 50
08 06 00 00 00 0C CD C9 23
```

其中宽度 `0x280 = 640`，被篡改后的高度 `0x250 = 592`。但尾部 CRC `0x0ccdc923` 恰好是 `640 × 640` IHDR 的正确校验值；若高度真为 592，CRC 应为 `0x1ef3d50c`。因此原始高度应同样是 `0x280`。

用十六进制编辑器把偏移 `0x14` 处的高度从：

```text
00 00 02 50
```

改为：

```text
00 00 02 80
```

原 CRC 无需再改。重新打开图片后，底部被隐藏的区域会显示出来：

![将 PNG IHDR 高度从 592 修复为 640 后，原先不可见的底部区域显现 flag](0xGame2020-week1-pcap-wp/restored-height-flag.png)

图中 flag 为 `0xGame{your_code_is_awesome}`。

## 方法总结

- 核心技巧：从 HTTP POST 中提取上传对象，再根据 PNG IHDR 与 CRC 的矛盾恢复真实高度。
- 识别信号：PCAP 中有少量 multipart 上传混在大量请求里；PNG 显示不完整且 IHDR CRC 校验失败。
- 复用要点：PNG 宽高是大端 32 位字段，IHDR CRC 覆盖 `IHDR` 类型和 13 字节数据；修改尺寸后必须验证 CRC 是否自洽。
