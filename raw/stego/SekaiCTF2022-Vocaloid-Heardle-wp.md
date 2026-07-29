# Vocaloid Heardle

## 题目简述

题目给出一段由多首 Vocaloid 短音频拼接而成的 MP3。生成脚本把 flag 花括号内的每个字符转成 Unicode 码点，并把该整数当作 Project SEKAI 的 `musicId`；随后从对应歌曲中截取前 3 秒，按字符顺序连接。

解题就是把音频每 3 秒切段，识别每段来自哪首歌，再把 `musicId` 用 `chr()` 转回字符。

## 解题过程

生成逻辑明确给出编码关系：

```python
flag = flag[6:-1]
tracks = [download(ord(char)) for char in flag]
```

歌曲主数据库的 `musicVocals.json` 把 `musicId` 映射到一个或多个 `assetbundleName`，短音频资源路径由该名称组成。生成器从每个 ID 的可选声部中随机选一个：

```python
def get_resource(music_id):
    choices = [
        item
        for item in resources
        if item["musicId"] == music_id
    ]
    return random.choice(choices)[
        "assetbundleName"
    ]
```

最终通过 FFmpeg 对每首歌执行：

```text
atrim=end=3
asetpts=PTS-STARTPTS
```

并按顺序 `concat`。所以边界固定在：

```text
0s, 3s, 6s, 9s, ...
```

可以人工听辨，也可以为候选短音频建立指纹。自动化流程为：

1. 下载主数据库中可能对应可打印 ASCII 的歌曲短音频；
2. 对参考音频和目标 3 秒片段统一采样率、声道数；
3. 提取频谱峰值指纹，或用滑动互相关计算相似度；
4. 为每段选择得分最高的候选 `musicId`；
5. 对匹配结果执行 `chr(music_id)`。

本题匹配出的 ID 为：

```text
118 48 67 97 108 111 73 100 60 51 117
```

转换后：

```python
ids = [
    118, 48, 67, 97, 108, 111,
    73, 100, 60, 51, 117,
]
print("".join(map(chr, ids)))
```

得到：

```text
v0CaloId<3u
```

补回格式：

```text
SEKAI{v0CaloId<3u}
```

## 方法总结

这类音频隐写的核心不是歌词，而是“选了哪段素材”。源码已经暴露一一映射 `字符 -> musicId -> 短音频`，因此无需通用语音识别；固定 3 秒边界和有限候选库让音频指纹或互相关足以稳定反解。外部歌曲链接只用于获取参考素材，编码规则与最终 ID 序列必须写入正文。
