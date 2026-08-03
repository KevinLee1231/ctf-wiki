# blackboxwarrior

## 题目简述

服务每轮从 CIFAR-10 测试集抽取一张 `224×224` 图片，要求上传一张平均哈希足够接近、但被隐藏 EfficientNet-B0 判成其他类别的图片。每成功一次，session 中的 `level` 加一；累计 50 次后返回 flag。若连续失败超过 3 次，进度会被清零。

仓库同时保留了部署文件 `challenge/site/app.py`、对外代码 `challenge/public/app.py` 和一份梯度攻击脚本。三者并不完全一致。Dockerfile 明确执行 `COPY site .`，因此判断真实赛题行为时应以 `site/app.py` 为准；该版本存在一个比对抗样本更直接的类型错误。公开参赛者记录也观察到了同一非预期解，并另外给出了平均哈希碰撞的预期思路：[UIUCTF 2022 Crypto + Misc Writeups](https://imp.ress.me/blog/2022-08-01/uiuctf-2022/#blackboxwarrior)。

## 解题过程

### 审计实际部署版本

`gen_img()` 把 CIFAR-10 数据集返回的数值标签直接存进 session：

```python
curr_image, label = image_set[random.randint(1, len(image_set))]
# label 是 0 至 9 的整数
return base64.b64encode(buffered.getvalue()).decode('utf-8'), label
```

而模型预测函数返回的是类别名称字符串：

```python
preds = torch.argmax(outputs, 1)

if hash_orig - hash_input < HASH_DIFFERENCE:
    return classes[preds]  # 如 "horse"、"ship"
else:
    return "no"
```

计分判断却直接比较二者：

```python
if pred != session['label'] and pred != 'no':
    session['level'] += 1
else:
    session['yolo'] += 1
```

即使模型正确识别了原图，也会出现类似：

```text
pred == "horse"
session["label"] == 7
```

字符串永远不等于整数。只要上传图片通过平均哈希检查，使 `pred != 'no'`，服务就把它当作分类改变并加分。最稳妥的输入正是服务器刚刚给出的原图：它尺寸、通道数和平均哈希都完全相同，不涉及任何随机分类结果。

### 连续回传原图

下面的脚本保留同一个 session，每轮从页面的 data URI 取出当前图片，再原样上传。`TARGET` 使用本地复现地址或赛事平台重新部署的地址，不应填写已经失效的旧比赛域名。

```python
import base64

import requests
from bs4 import BeautifulSoup

TARGET = "http://TARGET/"
session = requests.Session()

for attempt in range(60):
    page = session.get(TARGET, timeout=10)
    page.raise_for_status()

    soup = BeautifulSoup(page.text, "html.parser")
    src = soup.find("img")["src"]
    prefix = "data:image/png;base64,"
    assert src.startswith(prefix)
    image = base64.b64decode(src[len(prefix):])

    result = session.post(
        TARGET,
        files={"file": ("same.png", image, "image/png")},
        timeout=10,
    )
    result.raise_for_status()

    text = BeautifulSoup(result.text, "html.parser").get_text(" ")
    print(attempt + 1, text.strip())
    if "uiuctf{" in text:
        break
```

第 50 次成功时得到：

```text
uiuctf{oh_n0_my_b4nksy}
```

这条路径不需要加载 `model.pth`，也不需要知道模型把图片判成什么类别。

### 修正类型错误后的预期攻击

如果把 session 标签修成 `classes[label]`，原图就不会再通过。此时真正的约束是 `imagehash.average_hash()`：

1. 把图片转成灰度并缩放到 $8\times8$；
2. 计算 64 个像素的平均亮度；
3. 每个像素只记录“高于平均值”或“不高于平均值”这一位；
4. 两张图的距离只是这 64 位的汉明距离。

因此可以直接把 64 位哈希还原成黑白块图，再用最近邻放大到 `224×224`。生成图与原图的平均哈希完全相同，但对神经网络而言已经变成极不自然的块状图案，通常会落入另一个类别：

```python
import imagehash
import numpy as np
from PIL import Image

def make_hash_collision(challenge_image):
    digest = imagehash.average_hash(challenge_image)
    blocks = np.asarray(digest.hash * 255, dtype=np.uint8)

    crafted = Image.fromarray(blocks, mode="L")
    crafted = crafted.resize((224, 224), Image.Resampling.NEAREST)
    crafted = crafted.convert("RGB")

    assert imagehash.average_hash(crafted) - digest == 0
    return crafted
```

若某轮模型碰巧仍输出原类别，就会累计一次失败；超过 3 次失败会重置进度，所以完整客户端需要检测 `try again!` 并自动重试。仓库中的 `solvescript.py` 选择了另一条预期路线：加载相同模型，对输入做有目标梯度优化，尽量在保持图像接近的同时把输出推向 `frog`。两种方法的共同点都是攻击“感知相似度检查”和“模型分类边界”之间的不一致。

## 方法总结

- 首先区分实际部署代码、公开附件和意图说明。Dockerfile 的 `COPY site .` 证明真实路径使用整数标签；对外版本或官方 solver 只能用于解释预期设计，不能覆盖部署事实。
- 实际最短解是类型混淆：预测值为类别名字符串，真实标签为整数索引，导致相同图片也被视为错分。
- 修正逻辑后，平均哈希仍不是可靠的语义相似度。它把整张图压缩成 64 个阈值位，攻击者可以构造哈希相同、视觉和模型表征都完全不同的图片。
- 做 AI/ML CTF 时不应一上来就优化梯度；先检查数据预处理、标签表示、相似度门槛和成功条件，普通实现错误往往比模型攻击更决定性。
