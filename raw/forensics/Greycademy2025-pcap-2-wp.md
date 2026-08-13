# PCAP 2

## 题目简述

题目给出一次中间人抓取的 HTTP 流量，并提示需要“导出后在 Linux 执行”。PCAP 中包含网页资源和两个 ELF 文件，目标是掌握 Wireshark/TShark 的 HTTP object export，而不是只跟随一条文本流。

## 解题过程

用 Wireshark 的 `File → Export Objects → HTTP` 保存对象，或使用 TShark：

```bash
mkdir http-objects
tshark -r toddler.pcap --export-objects http,http-objects -q
file http-objects/*
```

导出结果中有 `RUN_THIS_IN_LINUX`、`DO_NOT_RUN_IN_LINUX` 和 `source.cpp`。其中 `RUN_THIS_IN_LINUX` 是 x86-64 PIE ELF；赋予执行权限后运行：

```bash
chmod +x http-objects/RUN_THIS_IN_LINUX
./http-objects/RUN_THIS_IN_LINUX
```

程序打印 “Very demure / Very mindful” 的 ASCII art。当前公开 PCAP 和导出对象中没有出现最终 flag 字符串；仓库 README 单独记录的提交值是：

```text
grey{3x724c71n9_f1135_15_345y}
```

因此本题的 HTTP 对象恢复链已经实测闭环，但公开材料没有说明 ASCII art 与该提交值之间是否还存在一层平台提示，不能把缺失映射写成已验证步骤。

## 方法总结

HTTP object export 会按响应实体重建文件，适合恢复下载的程序、图片和源码。导出后必须先用 `file` 定性再执行，不能只相信服务器给出的文件名；遇到公开附件与最终答案之间缺少映射时，应保留可验证恢复过程并明确资料边界。
