# 神秘的海报

## 题目简述

题目把 flag 分成两段：前半段作为文本写入 PNG 的 RGB 最低有效位，提取结果同时给出一份 WAV 音频的下载地址；后半段使用 Steghide 隐藏在音频中，并设置 6 位弱密码。需要依次完成 PNG LSB 提取和 WAV Steghide 解包。

## 解题过程

### 提取海报的 RGB 最低位

用 StegSolve 打开海报，进入 `Analyse -> Data Extract`，选择 Red、Green、Blue 三个通道的 bit 0，按行读取并采用 MSB First。也可以直接使用 `zsteg` 检查常见通道组合：

```bash
zsteg -a poster.png
```

提取出的关键信息为：

```text
This is part of the secret: hgame{U_Kn0w_LSB&W
I put the rest of the content here:
https://drive.google.com/file/d/13kBos3Ixlfwkf3e0z0kJTEqBxm7RUk-G/view?usp=sharing
This is my favorite music, there is another part of the secret in the music.
I use Steghide to encrypt, the password is also the 6-digit password we agreed at the time.
```

由此得到前半段：

```text
hgame{U_Kn0w_LSB&W
```

链接中的附件是 `Bossanova.wav`；即使不打开链接，以上文字也已经说明了文件类型、隐藏工具和密码特征。

### 从 WAV 提取后半段

题目明确提示密码是 6 位弱密码，常见值 `123456` 即可成功解包：

```bash
steghide info Bossanova.wav -p 123456
steghide extract -sf Bossanova.wav -p 123456
```

生成的 `flag2.txt` 内容为：

```text
恭喜你解到这里，剩下的Flag是 av^Mp3_Stego}，我们Week2见！
```

后半段为：

```text
av^Mp3_Stego}
```

拼接得到完整 flag：

```text
hgame{U_Kn0w_LSB&Wav^Mp3_Stego}
```

若不能直接猜中密码，可生成 `000000` 到 `999999` 的字典并用 `stegseek` 或循环调用 `steghide` 验证；成功标准是解包返回码为 0 且产生嵌入文件。

## 方法总结

- 核心技巧：先从 PNG 的 RGB bit 0 提取文本，再依据文本给出的工具、载体和密码特征处理第二层音频隐写。
- 易错点：StegSolve 要按 RGB、bit 0、Row、MSB First 组合提取；两段 flag 必须原样拼接。
- 复用要点：隐写输出不仅可能直接含 flag，也可能描述下一载体的来源、算法和口令，应把这些信息完整记录后再继续。
