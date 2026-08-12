# Hackergame2020 证验码 WP

## 题目简述

服务端生成一个由 62 种大小写字母和数字组成、长度为 16 的验证码，再把所有像素随机打乱。像素位置被彻底破坏，无法恢复原字符顺序；但提交界面只要求填写每种字符出现的次数，而随机置换不会改变颜色直方图。

附件给出了字体、字号、画布、字符集和渲染代码，因此可以为每个字符建立像素统计特征，再用最小二乘从混合直方图中估计字符计数。决定性步骤是从被随机打散的图像像素中提取隐藏字符多重集；这里没有模型训练、推断或对抗过程，因此归入隐写方向。

## 解题过程

### 把验证码改写成线性混合

字符使用同一字体独立绘制。忽略纯白背景后，把灰度值 $0\ldots254$ 的像素个数记为 255 维向量 $\operatorname{pix}(c)$。图像拼接和像素 shuffle 都不会改变计数，因此：

$$
\operatorname{pix}(c_1c_2\cdots c_{16})
=\sum_{c\in\mathcal A}n_c\operatorname{pix}(c),
$$

其中 $\mathcal A$ 是 62 字符字母表，$n_c$ 是字符 $c$ 的出现次数，且 $\sum n_c=16$。

依次渲染每个字符，把统计向量作为矩阵 $A\in\mathbb{R}^{62\times255}$ 的一行；把题目验证码的统计向量记为 $b\in\mathbb{R}^{255}$，未知计数为 $x\in\mathbb{Z}_{\ge 0}^{62}$，便有：

$$
A^\mathsf{T}x\approx b.
$$

### 去除彩色噪声并求最小二乘

验证码额外绘制了 10 条随机彩线。字体像素是灰度，即三个通道相等；统计时只保留 `R == G == B` 的像素，就能排除绝大多数彩色噪声。噪声线覆盖文字时仍会让部分灰度像素丢失，所以方程不是严格相等，使用最小二乘求近似解后四舍五入即可。

核心代码如下，其中 `img_generate`、`alphabet` 与字体文件均来自题目附件：

```python
import numpy as np

from shuffle import alphabet, img_generate


def count_gray(image):
    pixels = np.asarray(image.convert("RGB"))
    gray = (
        (pixels[:, :, 0] == pixels[:, :, 1])
        & (pixels[:, :, 1] == pixels[:, :, 2])
    )
    values = pixels[:, :, 0][gray]
    # bincount 的最后一格是纯白背景，舍去。
    return np.bincount(values, minlength=256)[:255]


def solve(shuffled_image):
    A = np.asarray([count_gray(img_generate(c)) for c in alphabet])
    b = count_gray(shuffled_image)
    floating, *_ = np.linalg.lstsq(A.T, b, rcond=None)
    counts = np.rint(floating).astype(np.int64)
    counts = np.clip(counts, 0, 16)
    assert counts.sum() == 16
    return dict(zip(alphabet, counts.tolist()))
```

用同一个 `requests.Session()` 先访问带 token 的首页，再请求 `/captcha_shuffled.bmp`。将 `solve()` 返回的计数按 `r_<字符>=<数量>` 作为参数提交到 `/result`。必须保持会话不变，因为验证码答案保存在 session 中。

在 shuffle 模式答对后，服务器不再打乱提示文本，返回形如：

```text
flag{Yuo-Cna-Raed-Tihs-<用户相关摘要>}
```

后缀由账号信息计算，每位选手不同。

## 方法总结

随机 shuffle 摧毁的是位置，不是多重集合统计量。只要验证目标也只依赖字符计数，就没有必要逆置换；已知渲染器使每个字符都能表示为固定特征向量，整个验证码自然变成一个超定线性混合问题。彩色噪声先用通道一致性过滤，剩余小误差再交给最小二乘处理，比尝试预测 `SystemRandom` 或训练图像识别模型更直接。
