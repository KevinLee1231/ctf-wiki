# conversations

## 题目简述

题目只提供一份 `capture.pcap`。流量中包含 HTTP 传输的多张 JPEG，正文并没有直接出现 flag；关键信息藏在其中一张图片的 EXIF 元数据里。决定性步骤是从抓包重组应用层对象，再检查导出文件的元数据。

## 解题过程

先让 `tshark` 按 HTTP 会话重组并导出对象：

```powershell
New-Item -ItemType Directory -Path './extracted_objects' -Force | Out-Null
tshark -r './capture.pcap' --export-objects 'http,./extracted_objects'
```

导出后不要只用图片查看器浏览像素内容，还要检查 JPEG 的 EXIF、注释和自定义字段。官方解法对所有 JPEG 运行 `exiftool` 并搜索比赛前缀：

```powershell
Get-ChildItem -LiteralPath './extracted_objects' -Filter '*.jpeg' | ForEach-Object {
    exiftool $_.FullName | Select-String -Pattern 'tjctf'
}
```

命中的元数据值为：

```text
tjctf{I_bh0p_to_sk00l_1337}
```

## 方法总结

- PCAP 中的文件应按协议重组，不能依赖对二进制抓包做 `strings`；分片、重传和编码都可能破坏直接搜索。
- 图片取证既包括像素，也包括 EXIF、注释、ICC、缩略图等容器字段。本题的视觉内容不是关键，因此没有把普通 JPEG 复制进 WP。
- `tshark --export-objects` 与 `exiftool` 组成了可复现的最短证据链：HTTP 对象恢复后，再从文件元数据定位 flag。
