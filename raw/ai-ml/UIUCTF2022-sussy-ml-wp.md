# Sussy ML

## 题目简述

附件提供 222 张编号为 `000.png` 至 `221.png` 的角色图片、一个残缺卷积网络的权重 `partial_sus_detector.pt`，以及负责加载模型的 `sussy.py`。原分类器的后半部分和分类头已经被破坏，只剩四层卷积与三次最大池化；题面同时说明异常样本比例低于 5%。

任务不是重新训练一个二分类器，而是利用残缺网络仍然保留的表征，把少数 “sus” 图片从大量正常图片中分离出来。公开参赛者题解也采用了“中间特征降维后寻找小簇”的路线；本文把其关键依据、完整代码和索引映射都写在正文中，外部记录仅作为结果交叉验证：[UIUCTF 2022 Crypto + Misc Writeups](https://imp.ress.me/blog/2022-08-01/uiuctf-2022/#sussy-ml)。

## 解题过程

### 理解残缺模型仍然输出了什么

附件中的前向传播为：

```python
def forward(self, x):
    x = F.relu(self.conv1(x))
    x = self.pooling(x)
    x = F.relu(self.conv2(x))
    x = self.pooling(x)
    x = F.relu(self.conv3(x))
    x = F.relu(self.conv4(x))
    x = self.pooling(x)
    return x
```

输入图片尺寸是 $32\times32$。三次 $2\times2$ 池化后，空间尺寸依次变为 $16\times16$、$8\times8$、$4\times4$；最后一层有 64 个通道，所以每张图片得到一个形状为 $64\times4\times4$、共 1024 维的特征，而不是 `sus/not sus` 两类概率：

```text
partial_outputs.shape == torch.Size([222, 64, 4, 4])
```

分类头虽然缺失，已训练卷积层仍会把语义相近的图片映射到相近区域。题面给出的异常比例小于 5%，意味着 222 个点中异常点少于约 $222\times0.05=11.1$ 个，可以把问题改写为无监督离群检测。

### 用 Isomap 把 1024 维特征压到一维

直接执行附件 `sussy.py` 会生成 `partial_outputs`。将每张图片的特征展平，再使用 Isomap 保留局部邻域关系：

```python
import numpy as np
from sklearn.manifold import Isomap

from sussy import partial_outputs

features = partial_outputs.detach().cpu().numpy()
features = features.reshape(len(features), -1)  # (222, 1024)

embedding = Isomap(n_components=1).fit_transform(features)[:, 0]
sus = np.flatnonzero(embedding > 0.035)
print(sus.tolist())
```

一维结果呈现一个大峰和一个明显分离的小峰。在原题环境中以 `0.035` 为分界，得到：

```text
[2, 31, 41, 59, 65]
```

注意阈值是 `0.035`。某些文字转述把它写成了 `0.35`，但公开代码、输出和题目 flag 元数据一致支持前者。

### 避免依赖坐标方向和固定阈值

降维坐标的正负方向本身没有语义，不同版本的数值尺度也可能略有变化。更稳妥的写法是在一维结果上分成两簇，再依据“异常比例小于 5%”选择较小的一簇：

```python
import numpy as np
from sklearn.cluster import KMeans
from sklearn.manifold import Isomap

from sussy import partial_outputs

x = partial_outputs.detach().cpu().numpy().reshape(222, -1)
z = Isomap(n_components=1).fit_transform(x)

labels = KMeans(
    n_clusters=2,
    random_state=0,
    n_init=20
).fit_predict(z)

counts = np.bincount(labels)
sus_cluster = counts.argmin()
sus = np.flatnonzero(labels == sus_cluster)

assert len(sus) / 222 < 0.05
flag = "uiuctf{" + "_".join(map(str, sorted(sus))) + "}"
print(flag)
```

`load_images()` 先对文件名排序，而文件名是补零的三位数字，所以特征张量的第 $i$ 行正好对应 `i.png`。最终输出为：

```text
uiuctf{2_31_41_59_65}
```

## 方法总结

- 分类头缺失不等于模型完全无用；卷积主干的中间激活仍是经过监督训练形成的特征空间，可用于聚类和离群检测。
- 题面中的“异常率低于 5%”是选择小簇的监督信息。它既限定异常数量，也能解决降维坐标正负翻转导致的固定阈值脆弱性。
- 必须核对样本顺序与文件编号的映射。本题使用补零文件名并在加载时排序，因此数组下标可以直接写入 flag；其他数据集不能默认成立。
- PCA、Isomap 或在特征上直接聚类都可能成功，关键不是某个特定算法，而是先把 1024 维表征转成能稳定区分“大多数正常样本”和“少数异常样本”的结构。
