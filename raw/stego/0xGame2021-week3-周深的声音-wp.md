# week3周深的声音

## 题目简述

附件通过多层载体隐藏数据：音频文件中使用 DeepSound 嵌入文本，文本内容是 Base64 编码的 JPEG，JPEG 内又通过 OutGuess 隐藏最终信息。

## 解题过程

先在 DeepSound 中打开音频附件并执行文件提取，得到 `flag.txt`。该文件的内容符合 Base64 字符集和长度特征，解码后保存为图片：

```bash
base64 -d flag.txt > decoded.jpg
```

确认输出文件具有 JPEG 文件头后，再使用 OutGuess 提取图片中的隐藏数据：

```bash
outguess -r decoded.jpg recovered.txt
```

其中 `-r` 表示从载体中恢复隐藏数据，第二个参数是输出文件。打开 `recovered.txt` 即可得到 flag。

## 方法总结

本题的关键是每一层都先识别真实数据类型，再选择下一步工具：DeepSound 负责音频容器、Base64 只负责表示层编码、OutGuess 负责 JPEG 隐写。不要因为中间文件名是 `flag.txt` 就停止分析，应以内容和文件头判断是否已经到达最终明文。
