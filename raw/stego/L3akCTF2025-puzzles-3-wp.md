# L3akCTF 2025 Puzzles 3 Writeup

## 题目简述

Puzzles 3 将每张图切成 $32\times32$，即 1024 个碎片，仍需完成 10 张图。这个规模已经不适合人工拖动，但 API 继续返回图片标题和原图来源，公开仓库也保留了 `build/images/level3` 下的全部原图。

本关的有效路线是复现服务端切图过程，以稀疏像素特征把 1024 个远端碎片一一映射回原始坐标。核心仍是空间碎片重组，因此归入 stego。

## 解题过程

### 建立本地模板

对响应中的 `title` 建立本地文件映射，例如：

```python
IMAGE_PATHS = {
    "Forest Adventure": "images/level3/adventure.png",
    "City Lights": "images/level3/city_lights.png",
    "One Day I Will be Healed": "images/level3/healed.png",
    "Restricted Area": "images/level3/restricted_area.png",
    "Are You Ready For Your Biggest Adventure Yet?":
        "images/level3/big_adventure.png",
    "The Final Battle": "images/level3/battle.png",
    "今日の日をいつか 思い出せ": "images/level3/umbreon.png",
    "Grani in the City": "images/level3/grani.png",
    "City Night": "images/level3/cyberpunk.png",
    "Studying for Apocraphy Exam": "images/level3/apocraphy.png",
}
```

服务端边框宽度为 5。按源码公式裁剪、补透明边框，再以 $32\times32$ 切分：

```python
from PIL import Image


def make_reference_tiles(path, rows=32, cols=32, border=5):
    source = Image.open(path).convert("RGBA")
    body_width = ((source.width + 2 * border) // cols) * cols - 2 * border
    body_height = ((source.height + 2 * border) // rows) * rows - 2 * border
    source = source.crop((0, 0, body_width, body_height))

    padded = Image.new(
        "RGBA",
        (body_width + 2 * border, body_height + 2 * border),
        (0, 0, 0, 0),
    )
    padded.paste(source, (border, border))

    tile_width = padded.width // cols
    tile_height = padded.height // rows
    return [
        padded.crop((
            x * tile_width,
            y * tile_height,
            (x + 1) * tile_width,
            (y + 1) * tile_height,
        ))
        for y in range(rows)
        for x in range(cols)
    ]
```

### 控制特征规模

若对每一对切片逐像素比较，10 张图片会产生大量无意义计算。官方脚本只在每个方向大约采样 5 次：

```python
def sample_pixels(image, attempt=0):
    step_y = max(1, image.height // 5 - attempt * 3)
    step_x = max(1, image.width // 5 - attempt * 3)
    return [
        image.getpixel((x, y))
        for y in range(0, image.height, step_y)
        for x in range(0, image.width, step_x)
    ]
```

第一次仅使用稀疏特征；若提交失败，增加 `attempt` 会缩短步长并采集更多像素。这样把高精度计算只留给确实存在重复纹理的图片。

### 计算一对一映射

从 API 返回的 Base64 数据解码 1024 个 PNG：

```python
import base64
import io

from PIL import Image


def decode_piece(value):
    data = base64.b64decode(value)
    return Image.open(io.BytesIO(data)).convert("RGBA")
```

对参考块和远端块的采样像素计算曼哈顿距离。参考透明像素表示外框，只检查对应远端位置是否也是浅色边框；图像主体正常累计 RGB 差值：

```python
def distance(left, right):
    total = 0
    for p1, p2 in zip(left, right):
        if p1[3] < 10:
            if min(p2[:3]) < 127:
                total += 1000
        elif p2[3] < 10:
            if min(p1[:3]) < 127:
                total += 1000
        else:
            total += sum(abs(a - b) for a, b in zip(p1[:3], p2[:3]))
    return total
```

按原始行优先位置遍历参考块，每次从尚未使用的远端块中选择距离最小者：

```python
unused = set(range(1024))
answer = []

for reference in reference_features:
    candidate = min(
        unused,
        key=lambda index: distance(reference, received_features[index]),
    )
    answer.append(candidate)
    unused.remove(candidate)
```

这一步的最坏比较次数约为 $1024^2/2$，但每次只比较几十个采样点，单张图仍可在时限内完成。答案数组是“原始位置到返回索引”的映射，不要再做一次逆置换。

### 处理限速和误配

服务端对 `/api/newpuzzle`、`/api/checkanswer` 等接口设置 100 毫秒限速。官方脚本在取得拼图后至少等待约 350 毫秒再提交，避免求解很快时反而收到 `rate limited`：

```python
elapsed = time.time() - started
time.sleep(max(0, 0.350 - elapsed))
```

若返回 `correct: false`：

1. 保存当前拼接结果辅助定位重复纹理；
2. 调用正确的 `/api/skippuzzle` 路由删除当前实例；
3. 增大采样密度；
4. 获取同一关的新拼图并重试。

完成 10 张 $32\times32$ 拼图后，请求零基关卡编号 `2`：

```json
{"level": 2}
```

得到：

```text
L3AK{3_th3_ar7w0rk_1s_pr3tty_c00l_i_7h1nk}
```

## 方法总结

Puzzles 3 的难点主要是规模，而不是出现了新的图像原语。利用已知原图后，问题仍是模板切片与远端切片之间的一对一最近邻匹配。复现裁剪边界、正确屏蔽随机外框、控制采样密度，比换用复杂的通用计算机视觉模型更重要。

对于大规模碎片匹配，应先使用廉价特征完成绝大多数确定映射，再只对失败样本提高精度。还要把网络限速计入 solver：本地算法越快，越容易在缺少显式节流时被应用层拒绝。
