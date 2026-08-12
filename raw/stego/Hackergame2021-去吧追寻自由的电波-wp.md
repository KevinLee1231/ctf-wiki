# 去吧！追寻自由的电波

## 题目简述

附件是一段能够播放、但语速和音高明显异常的 MP3。有效信息并未经过复杂加密，而是通过错误的采样率解释隐藏在人声中；恢复正常语速后，还需要把 NATO 音标字母表读出的单词映射回字母。

## 解题过程

### 恢复播放速度

原音频标称采样率为 48000 Hz。把同一批样本按较低采样率解释，播放速度和音高会同步下降。使用 FFmpeg 将解释采样率改为 24000 Hz 即可听清，若设备上仍偏快可尝试 16000 Hz：

```bash
ffmpeg -i radio.mp3 -af "asetrate=24000" radio-slow.mp3
```

`asetrate` 改的是样本的时间解释，不是单纯做保持音高的变速，因此正好抵消题目对采样率的伪装。

### 解读音标字母

减速后听到的内容为：

```text
Foxtrot Lima Alfa Golf left-bracket
Papa Hotel Oscar November Echo Tango India Charlie Alfa Bravo
right-bracket
```

按 NATO 音标字母表取每个单词代表的首字母，并把 `left-bracket`、`right-bracket` 分别还原为 `{`、`}`，得到：

```text
flag{phoneticab}
```

## 方法总结

- 核心技巧：通过修改采样率恢复被整体加速、升调的人声，再解码 NATO 音标字母。
- 识别信号：声音内容像人声但整体速度与音高同时异常，通常说明样本率解释错误。
- 复用要点：不要先做复杂频谱分析；先比较 1/2、1/3 等简单采样率比例。恢复后仍像拼读时，再检查音标字母表或逐字符口令。
