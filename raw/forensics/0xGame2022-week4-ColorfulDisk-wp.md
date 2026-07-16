# week4ColorfulDisk

## 题目简述

题目附件是一份磁盘镜像。镜像中包含一个加密容器、一个加密压缩包和 `password.txt`。需要先恢复并挂载容器，再从其中图片的 RGB 像素字节提取音频，解码 SSTV 图像得到压缩包密码，最后取出 flag。

## 解题过程

先把磁盘镜像加载到取证工具中，按文件类型与加密文件分类检查。镜像里可见加密文件 `1`、`flag.zip`，以及提供第一阶段口令或挂载提示的 `password.txt`。

| 文件 | 识别结果 | 在解题链中的作用 |
| --- | --- | --- |
| `1` | TrueCrypt 加密容器，算法栏显示 AES/Twofish/Serpent | 挂载后取得异常 PNG |
| `flag.zip` | WinZip AES-256 加密压缩包 | 使用 SSTV 中恢复的密码解压 |
| `password.txt` | 普通文本文件 | 提供第一阶段挂载口令或提示 |

导出这些文件，按照 `password.txt` 的提示挂载加密容器 `1`，可得到一张尺寸为 $1042\times1042$ 的异常 PNG。题目提示强调 RGB，说明有效数据不是按位藏在某个通道中，而是把图像每个像素的 R、G、B 三个通道依次连接成原始字节流。

Pillow 的 `tobytes()` 默认按从左到右、从上到下的 RGB 顺序输出，正好等价于逐像素、逐通道提取：

~~~python
from pathlib import Path

from PIL import Image

image = Image.open("1.png").convert("RGB")
assert image.size == (1042, 1042)
Path("hidden.wav").write_bytes(image.tobytes())
~~~

输出文件以 WAV 头开始，可以正常播放。音频中的规律扫频是 SSTV（慢扫描电视）信号：它用不同音频频率表示同步信号、亮度和颜色，并按扫描线传输一幅静态图像。把 `hidden.wav` 输入 SSTV 解码器后，图像中显示第二阶段密码：

~~~text
ty#48u*k
~~~

使用该密码解压 `flag.zip`，得到：

~~~text
0xGame{RGB_C0lor_Pix3l}
~~~

## 方法总结

完整证据链是“磁盘镜像 → 加密容器 → PNG 像素字节 → WAV → SSTV 图像 → 压缩包”。遇到看似噪声的图片时，应结合尺寸、提示词和导出字节的文件头判断数据组织方式；这里必须保留 RGB 通道顺序和行优先顺序，否则无法还原可识别的 WAV。
