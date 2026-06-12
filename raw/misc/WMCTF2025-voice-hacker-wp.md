# Voice_hacker

## 题目简述

题目是一个语音认证系统，页面提示用户点击录音按钮并说出“CTF，启动！”。抓包后可以看到认证接口为 `/api/authenticate`，上传字段名是 `audio`，文件类型为 wav。附件流量里还有一段 RTP 语音数据，需要从 UDP 流中恢复出目标音频，再伪造一个能通过认证的 wav 发送给接口。

后端源码里还有一个关键点：前端按钮并不会真的提交麦克风录音，而是调用 `/api/fake_audio` 获取随机 wav，再提交给 `/api/authenticate`。认证接口读取 `/app/seed.wav` 和上传音频，只比较音频长度、平均振幅、最大振幅、样本数等简单特征，综合相似度达到 90% 即返回 flag。因此这题的核心不是语音识别文本，而是从流量恢复目标音频特征，再构造足够接近的 wav。

附件中的后端入口是 `docker/backend/src/main.py`，核心逻辑在 `src/routes/voice_auth_simple.py`。`/api/fake_audio` 会生成随机音频：采样率从 `22050`、`48000` 中选，时长落在 `0.6-1.0s` 或 `2.8-3.2s`，频率在 `1800-4000Hz`，还会叠加强噪声，目的就是让页面按钮默认提交的音频很难误过。真正可控的接口仍是 `/api/authenticate` 的 `audio` 表单字段。

## 解题过程

打开题目链接后，页面显示语音认证系统，要求说出“CTF，启动！”：

![打开题目链接。是一个音频认证系统，要求说出CTF，启动通过认证来获取flag](<WMCTF2025-voice-hacker-wp/打开题目链接-是一个音频认证系统-要求说出ctf-启动通过认证来获取flag.png>)

通过浏览器抓包可以看到认证请求会把音频发到 `/api/authenticate`。这个接口需要先记住，后面可以绕过页面逻辑直接 POST 自己生成的 wav：

![通过抓包可以发现会发送一个音频出去，那个认证接口需要先记住](<WMCTF2025-voice-hacker-wp/通过抓包可以发现会发送一个音频出去-那个认证接口需要先记住.png>)

打开提供的流量文件，主体是一批 UDP 包：

![然后打开提供的流量文件，发现是一堆udp的数据](<WMCTF2025-voice-hacker-wp/然后打开提供的流量文件-发现是一堆udp的数据.png>)

结合包头特征和端口可以判断这是 RTP 流量。RTP 常用于 VoIP/电话音频传输，Wireshark 默认未必能自动识别，所以需要对这批 UDP 执行 `Decode As...`，协议选择 RTP：

![流量头特征和接收的端口可以知道这是个RTP的流量，这个流量一般由于电话协议，这里我们decode as把udp流量选择为RTP](<WMCTF2025-voice-hacker-wp/流量头特征和接收的端口可以知道这是个rtp的流量-这个流量一般由于电话协议-这里我们decode-as把udp流.png>)

随后在 Wireshark 的 RTP 分析/播放功能中导出音频。导出的音频就是后续构造认证音频的参考样本：

![然后电话中RTP流拿到音频](<WMCTF2025-voice-hacker-wp/然后电话中rtp流拿到音频.png>)

![然后电话中RTP流拿到音频](<WMCTF2025-voice-hacker-wp/然后电话中rtp流拿到音频-06.png>)

一种稳妥做法是把提取出的参考音频交给 GPT-SoVITS 这类少样本语音克隆/TTS 工具处理。GPT-SoVITS 的作用是：用较短参考语音学习音色，再按给定文本合成新的 wav；本题中目标文本就是“CTF，启动！”。工具不唯一，RVC、SoVITS 或其他能控制音色与文本的 TTS 都可以。保留项目地址作为来源：<https://github.com/RVC-Boss/GPT-SoVITS>。

![将音频提取后交给GPT-SoVITS](<WMCTF2025-voice-hacker-wp/将音频提取后交给gpt-sovits.png>)

![将音频提取后交给GPT-SoVITS](<WMCTF2025-voice-hacker-wp/将音频提取后交给gpt-sovits-08.png>)

![将音频提取后交给GPT-SoVITS](<WMCTF2025-voice-hacker-wp/将音频提取后交给gpt-sovits-09.png>)

从源码角度看，后端最终比较的是简单 wav 特征：

```python
duration_sim = 1 - abs(features1["duration"] - features2["duration"]) / max(features1["duration"], features2["duration"])
amp_sim = min(features1["avg_amplitude"], features2["avg_amplitude"]) / max(features1["avg_amplitude"], features2["avg_amplitude"])
sample_sim = 1 - abs(features1["sample_count"] - features2["sample_count"]) / max(features1["sample_count"], features2["sample_count"])
similarity = duration_sim * 0.4 + amp_sim * 0.3 + sample_sim * 0.3
```

源码里还会在上传音频时长处于 `0.5-3.0s` 时加 `0.05`，平均振幅大于 0 时再加 `0.05`。所以生成音频时要注意时长、采样率、音量不要和参考样本差太远。生成 `audio.wav` 后，用前面抓到的接口直接提交：

```python
import requests

url = "http://<target>/api/authenticate"

wav_path = r"audio.wav"

headers = {
    "Accept-Language": "zh-CN,zh;q=0.9",
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36",
    "Accept": "*/*",
    "Origin": "http://<target>",
    "Referer": "http://<target>/",
    "Accept-Encoding": "gzip, deflate, br",
    "Connection": "keep-alive"
}

with open(wav_path, "rb") as f:
    files = {
        "audio": ("recording.wav", f, "audio/wav")
    }
    response = requests.post(url, headers=headers, files=files)

print("Status Code:", response.status_code)
print("Response Text:", response.text)
```

![最后输出“C T F，启动！”生成伪造音频即可，然后这里用上前面的接口，直接发送wav过去](<WMCTF2025-voice-hacker-wp/最后输出-c-t-f-启动-生成伪造音频即可-然后这里用上前面的接口-直接发送wav过去.png>)

响应中 `success` 为 `true` 时会返回 flag。源码中的成功分支如下：

```python
if similarity_percentage >= 90.0:
    return jsonify({
        "success": True,
        "message": "认证成功",
        "flag": "WMCTF{01a9a4f1-e748-43fa-8d6d-bba372016adc}"
    })
```

## 方法总结

- 核心技巧：从 UDP 流量中识别 RTP，导出目标音频，再构造与参考音频特征足够接近的 wav。
- 识别信号：题目是“语音认证”，但页面录音逻辑和后端认证逻辑不一致；看到 `/api/fake_audio`、`/api/authenticate`、`seed.wav`、简单特征相似度时，应优先走接口伪造而不是只在浏览器里录音。
- 复用要点：AI TTS/语音克隆工具只是生成音频的一种方式，真正要满足的是后端比较的特征。提交前应检查时长、采样率、平均振幅和样本数，避免生成内容听起来正确但特征相似度不足。
