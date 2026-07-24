# UMDCTF 2019 - SecretSounds

## 题目简述

附件被识别为 Sun/NeXT `.au` 音频：8 位线性 PCM、单声道、2000 Hz。题面称有人把工具伪装成音频文件，实际载荷藏在 24 字节 AU 头之后的样本数据中。

## 解题过程

AU 头给出的数据偏移是 `0x18`。音频数据开头为：

```text
ff c5 cc c6 ...
```

每字节异或 `0x80` 后变成 `7f 45 4c 46`，即 ELF magic。按这个关系恢复整个样本区：

```python
from pathlib import Path

audio = Path("SecretSounds").read_bytes()
data_offset = int.from_bytes(audio[4:8], "big")
payload = bytes(value ^ 0x80 for value in audio[data_offset:])
Path("recovered-elf").write_bytes(payload)
```

恢复文件是 ELF，同时也是带有附加 7z 数据的自解压程序。无需执行未知程序，直接让 7-Zip读取归档并提取图片：

```bash
7z l recovered-elf
7z x recovered-elf -oextracted
```

归档中的 `flag.png` 显示：

![从 AU 样本异或恢复的 ELF 自解压归档内的 flag 图片](./UMDCTF2019-secretsounds-wp/recovered-flag-banner.png)

```text
UMDCTF-{MUSIC_TO_MY_EARS}
```

其 SHA-256 与官方摘要一致。

## 方法总结

8 位 PCM 的符号位翻转可以让任意字节流听起来像音频数据，同时用固定异或恢复。分析时应读取容器声明的数据偏移，并检查简单变换后的 magic。恢复出的可执行文件也不必直接运行；若尾部是已知归档格式，静态列目录和解包更安全。
