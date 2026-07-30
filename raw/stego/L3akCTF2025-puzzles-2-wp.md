# L3akCTF 2025 Puzzles 2 Writeup

## 题目简述

Puzzles 2 把每张原图切成 $12\times12$，单张图共有 144 个碎片，并要求完成本关全部 10 张图片。手工拖动理论上可行，但耗时很长；题目响应会给出图片标题、作者和原图地址，意图是让选手取得原图后自动匹配切片。

本关没有 Web 漏洞，成功条件仍是恢复图片的二维排列。决定性障碍是大规模图像碎片重组，因此归入 stego。

## 解题过程

### 获取当前拼图和对应原图

保持同一会话访问：

```http
GET /api/newpuzzle
Cookie: session=...
```

响应中的关键字段为：

```json
{
  "title": "Bodega Catgirl",
  "url": "https://原图来源",
  "pieces": ["base64 PNG", "..."],
  "puzzle_id": "...",
  "rows": 12,
  "cols": 12,
  "width": 123,
  "height": 123
}
```

根据 `title` 或 `url` 下载相应原图。公开仓库的 `build/images/level2` 已包含服务实际使用的 10 张原图，因此离线复现时无需依赖已经失效的社交媒体链接。

### 精确复现服务端裁剪

服务端先把原图裁成加入两侧边框后能被行列数整除的尺寸。设原始宽高为 $W,H$，边框宽度 $b=5$，列数和行数均为 12，则主体尺寸为：

$$
W'=\left\lfloor\frac{W+2b}{12}\right\rfloor\cdot12-2b
$$

$$
H'=\left\lfloor\frac{H+2b}{12}\right\rfloor\cdot12-2b
$$

随后在四周补 5 像素随机浅色边框，再切成 144 块。官方脚本在本地使用透明边框，并在距离函数中把透明像素当成外框掩码，从而不需要预测随机颜色。

```python
from PIL import Image


def split_reference(path, rows=12, cols=12, border=5):
    image = Image.open(path).convert("RGBA")
    width = ((image.width + 2 * border) // cols) * cols - 2 * border
    height = ((image.height + 2 * border) // rows) * rows - 2 * border
    cropped = image.crop((0, 0, width, height))

    padded = Image.new(
        "RGBA",
        (width + 2 * border, height + 2 * border),
        (0, 0, 0, 0),
    )
    padded.paste(cropped, (border, border))

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

### 用稀疏像素指纹匹配

无需比较每个像素。对每块图按横纵方向约 5 个采样点建立指纹：

```python
def fingerprint(image, samples=5):
    step_x = max(1, image.width // samples)
    step_y = max(1, image.height // samples)
    return [
        image.getpixel((x, y))
        for y in range(0, image.height, step_y)
        for x in range(0, image.width, step_x)
    ]
```

对每个原始位置，计算它与所有尚未使用的远端块之间的 RGB 差值，选择最小者，并立即从候选池移除。透明参考边框与远端浅色边框之间不计算普通 RGB 差值；如果透明位置对应远端深色主体，才施加较大惩罚。

```python
def score(reference, received):
    value = 0
    for left, right in zip(reference, received):
        if left[3] < 10:
            value += 1000 if min(right[:3]) < 127 else 0
        elif right[3] < 10:
            value += 1000 if min(left[:3]) < 127 else 0
        else:
            value += sum(abs(a - b) for a, b in zip(left[:3], right[:3]))
    return value


unused = set(range(144))
answer = []

for reference in reference_fingerprints:
    index = min(
        unused,
        key=lambda candidate: score(
            reference,
            received_fingerprints[candidate],
        ),
    )
    answer.append(index)
    unused.remove(index)
```

最终 `answer[i]` 表示原始位置 $i$ 对应 `pieces` 数组中的哪一块。提交：

```http
POST /api/checkanswer
Content-Type: application/json

{"puzzle_id":"当前 UUID","answer":[...144 个索引...]}
```

若个别图片出现近似区域导致误配，可以提高采样密度后重试；错误拼图应调用 `/api/skippuzzle` 跳过，再取得该图片的新洗牌。

完成 10 张图后，请求关卡编号 `1`，得到：

```text
L3AK{2_i_5ur3_hop3_u_d1dn7_d0_th4t_by_h4nd}
```

## 方法总结

本关从“看边拼图”转成了有已知模板的图像配准问题。最重要的工程细节是复现裁剪尺寸和外框位置；若切片边界相差一个像素，即使使用完整原图，所有距离也会明显增大。

处理 144 块时，稀疏像素指纹、一对一贪心匹配和失败后增加采样密度已经足够。相比通用拼图求解器，利用题目主动泄露的原图标题与地址，搜索空间会从排列组合直接降为 144 次局部最近邻匹配。
