# 每日推荐

## 题目简述

附件是一份浏览 WordPress 时产生的 PCAP。HTTP 对象列表中有一个约 8.29 MB 的 `multipart/form-data` 上传远大于其他请求；从中可雕取加密 ZIP，解出 MP3 后还要查看频谱图中的可视化文字。PCAP 提取只是取得载体，决定性信息藏在音频频谱中，因此归入 Stego。

## 解题过程

在 Wireshark 中应用 `http` 过滤器并打开“文件 → 导出对象 → HTTP”，按大小排序。异常对象对应 `async-upload.php`，内容类型为 `multipart/form-data`，约 8290 kB。保存该请求体后检查十六进制数据，可以看到上传字段和文件名：

```text
Content-Disposition: form-data; name="async-upload"; filename="song.zip"
Content-Type: application/x-zip-compressed

50 4b 03 04 ...
```

从 `50 4b 03 04` 开始切出 ZIP，或让 `binwalk`、`foremost` 自动雕取。压缩包内是 `I Love Mondays.mp3`，注释提示密码为 6 位数字。遍历 `000000` 到 `999999` 后得到：

```text
759371
```

解出 MP3 后，正常播放能听到明显异常，但 flag 不是普通语音。把音频切换到频谱视图，可以直接读出被绘制在频率能量上的文字：

![MP3 频谱中清晰绘制出的 hgame I love EDM233 字样](./HGAME2020-每日推荐-wp/edm-spectrogram.png)

按比赛格式整理为：

```text
hgame{I_love_EDM233}
```

## 方法总结

- 核心技巧：从 PCAP 中定位体积异常的 HTTP 上传，雕取嵌套 ZIP，再观察音频频谱而非只听波形。
- 识别信号：对象列表中存在显著离群的大请求、multipart 字段含文件名、MP3 听感异常或题面暗示音乐。
- 复用要点：HTTP 请求体包含 MIME 边界和头部，切 ZIP 时必须从真实 `PK` 文件头开始；短数字口令可以穷举，但应先利用注释缩小字符集与长度。
