# Hackergame2020 来自未来的信笺 WP

## 题目简述

附件解压后包含 351 张二维码图片。题面中的 “Send from Arctic” 对应 GitHub Archive Program 的北极代码存档：每张二维码不是一段可读文本，而是归档文件的一段原始二进制数据。

本题的主障碍是按正确顺序解码并重组被分散到大量二维码中的隐藏载荷，因此归入隐写方向。

## 解题过程

先按文件名的字典序排列 `frame-*.png`。这些二维码使用 Version 40、L 级纠错，尺寸为 $177\times177$，单张最多承载 2953 字节的二进制数据。普通在线识别器常把内容当字符串处理，遇到 `0x00` 会截断；旧版 ZBar 还会猜测字符编码，导致输出与二维码中的原始字节不同。

使用 ZBar 0.23.1 或更高版本，并同时指定 `--raw` 与 `-Sbinary`，可以关闭文本包装并按二进制模式输出。Shell 的 glob 会按字典序展开这些固定格式文件名，因此可以直接顺序追加：

```sh
#!/bin/sh
set -eu

unzip frames.zip -d frames
: > repo.tar

for frame in frames/frame-*.png; do
    zbarimg --raw -q -Sbinary "$frame" >> repo.tar
done

tar -xf repo.tar
tar -xJf repo.tar.xz
```

第一层重组结果是一个 tar 归档，其中包含：

- `META`：归档元数据；
- `COMMITS`：提交信息；
- `repo.tar.xz`：真正的仓库内容。

继续解压 `repo.tar.xz`，再在还原出的仓库中读取 flag 文件即可完成题目。必须使用二进制重定向逐段拼接，不能复制二维码识别器显示的文本，否则任意一次截断或编码转换都会破坏 tar 结构。

## 方法总结

大量二维码通常不是让人逐张阅读，而是在模拟一种离线存储或分片传输协议。解题关键有三点：保持文件排序稳定、让解码器输出原始字节、以二进制方式无缝拼接。验证时可先用 `file repo.tar` 或 `tar -tf repo.tar` 检查外层归档，再逐层解包，能快速定位是解码、排序还是拼接环节出错。
