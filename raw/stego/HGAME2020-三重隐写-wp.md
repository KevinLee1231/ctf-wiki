# 三重隐写

## 题目简述

附件把三条密钥链分散在 MP3 封面、WAV 的 LSB 和另一首 MP3 的 Stego 载荷中。三条线索分别给出 Encrypto 的 AES 密钥、MP3Stego 密钥和 7z 密码；按顺序提取后才能解压并解密 `flag.crypto`。

## 解题过程

首先检查 `Unlasting.mp3` 的内嵌封面。封面中间是一条 PDF417 二维条码，属于有明确视觉结构的信息，应保留而不是只描述为“某张截图”：

![Unlasting.mp3 封面中嵌入的横向 PDF417 条码](./HGAME2020-三重隐写-wp/unlasting-cover-pdf417.png)

用支持 PDF417 的扫码器读取后得到：

```text
AES key: 1ZmmeaLL^Typbcg3
```

这把密钥对应附件中的 `flag.crypto`。`.crypto` 是 Encrypto 工具生成的加密文件，后续应使用同一工具和该 AES key 解密。

第二个音频名为 `You know LSB.wav`，文件名已经明确提示最低有效位。用 SilentEye 打开 WAV 并选择 Decode，可恢复：

```text
Stegano key: uFSARLVNwVIewCY5
```

第三个 MP3 的文件名为 `上裏与手抄卷.mp3`。结合 “Stegano” 提示，使用 MP3Stego，并把刚得到的密钥作为 `-P` 参数：

```powershell
Decode.exe -X -P uFSARLVNwVIewCY5 "上裏与手抄卷.mp3" out.wav hide.txt
```

`hide.txt` 内容为：

```text
Zip Password: VvLvmGjpJ75GdJDP
```

用这串密码解开 `flag.7z`，取出 `flag.crypto`；再用 Encrypto 和第一步得到的 `1ZmmeaLL^Typbcg3` 解密，最终得到：

```text
hgame{i35k#zIewynLC0zfQur!*H9V$JiMVWmL}
```

## 方法总结

- 完整链路：MP3 封面 PDF417 得 AES key，WAV LSB 得 MP3Stego key，MP3Stego 得 7z 密码，最后解压并用 Encrypto 解密。
- 关键细节：三串密钥用途不同，必须依据线索文字和输出标签对应到正确载体，不能互换尝试后只记录“某密码可用”。
- 图片取舍：工具窗口和终端结果均已转写为正文；只保留承载条码空间结构的封面图，并使用能说明内容的文件名与替代文本。
