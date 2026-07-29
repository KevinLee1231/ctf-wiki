# Mind Calculator

## 题目简述

服务端通过 WebSocket 连续发送英语语音算式。每题包含 10 个 $[-9999,9999]$ 内的整数，第一项直接加入结果，后九项随机使用加号或减号。语音由 Edge TTS 从多种英语口音中随机选择一个生成，客户端必须在每轮 25 秒内识别、计算并回传答案。

连接后先发送文本 `start`。服务端会把计数器从 `-1` 更新到 `0`，之后需要答对 100 道算式，且错误次数必须少于 20，才能收到 flag。

## 解题过程

WebSocket 一轮响应中可能先出现文本状态，再分多帧发送 MP3，最后以空二进制帧结束。客户端应持续拼接二进制帧，而不是把单个网络帧当成完整音频：

```python
audio = b""
while True:
    message = ws.recv()
    if isinstance(message, str):
        print(message)
        continue
    if not message:
        break
    audio += message
```

官方解法先用 FFmpeg 把 MP3 统一成 Vosk 要求的 16 kHz、单声道、16 位 PCM WAV，并去掉首尾静音：

```text
ffmpeg -i a.mp3 -acodec pcm_s16le -ac 1 -ar 16000 \
  -af silenceremove=1:0:-50dB -y a.wav
```

通用语音识别容易把数字短语自动改写或误分词。这里的词汇域很小，可以给 Vosk 的识别器提供受限语法，重点保留：

```text
what is negative plus minus and
one ... nineteen
twenty thirty ... ninety
hundred thousand
```

官方方案使用 `vosk-model-en-us-0.22-lgraph`；本地模型约需 5 秒一题，记录的准确率约为 85%。Azure Speech 方案约需 20 秒一题，准确率约为 92%，仍在单题 25 秒限制内。模型下载页可以保留为工具入口：[Vosk Models](https://alphacephei.com/vosk/models)。

识别结果不要直接交给表达式求值器，而应按运算符切成三元组：

```text
(what is | plus | minus)
(可选的 negative)
(数字词组)
```

数字词组可由 `word2number` 转成整数。操作符 `minus` 和数字自身的 `negative` 必须分别处理；例如 “minus negative five” 的贡献是 $+5$：

```python
calculated = 0
for operator, sign, words in terms:
    value = word_to_num(words)
    if operator == "minus":
        value *= -1
    if sign == "negative":
        value *= -1
    calculated += value
```

识别失败时仍应及时发送一个占位答案，让服务端进入下一题；20 次容错足以覆盖少量口音误识别。达到 100 个正确答案后得到：

```text
SEKAI{Srsly,_u_didnt_do_this_manually,_...or_did_U?}
```

## 方法总结

本题的主要障碍不是算术，而是低延迟、有限词表的语音识别流水线。稳定方案包括正确重组 WebSocket 音频帧、规范化音频格式、用受限语法降低识别空间，以及把“运算符负号”和“数字自身负号”分开解析。外部 ASR 服务只是模型选择，完整协议和计算逻辑均可从题目源码离线复现。
