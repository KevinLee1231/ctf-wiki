# week4Strobe Memory

## 题目简述

这是一道 Windows 7 内存取证题。需要从内存镜像中恢复用户口令和命令行历史，再按题面指定的 RC4 解开一段 OpenSSL 格式密文；解密结果给出一个用于 Tupper 公式绘图的大整数，最终图像中包含 flag。

## 解题过程

先用 Volatility 2 识别镜像版本：

```bash
python vol.py -f memdump.mem imageinfo
```

候选配置中可确定系统为 `Win7SP1x64`，后续命令显式指定该 profile。题目要求寻找登录口令，可以使用 `hashdump` 提取 NT hash 后离线恢复，也可以使用 Volatility 的 Mimikatz 插件直接检查内存中的凭据：

```bash
python vol.py -f memdump.mem --profile=Win7SP1x64 mimikatz
```

得到口令：

```text
ffxiv
```

继续查看进程命令行：

```bash
python vol.py -f memdump.mem --profile=Win7SP1x64 cmdline
```

命令行历史中存在一个云盘链接，下载后得到以 `U2FsdGVkX1` 开头的 Base64 文本。这个前缀解码后是 OpenSSL 的 `Salted__` 文件头，只能说明密文带有 OpenSSL 盐值，不能据此判断它使用 AES。根据题目提示选择 RC4，并使用刚恢复的 `ffxiv` 作为口令解密，得到关键词 `tupper` 和下面这个大整数 `$k$`：

```python
k = int(
    "12759973993396709313115245209296720371500426881932460187720370869509290553987976"
    "62543803518241194333094506052196626697053228790392884060796858947008768539393435"
    "46466674167675843632299107885531659513226127945849860968719044332851963670087790"
    "02528368527914868356610182254161654881601408901868157066537667875257763930515589"
    "00476802704127891703153310237084375088908048432974181678093798688698383236583373"
    "05551758361448088050669051796954650620349250879619020031969406174600372916970856"
    "329970323239288072622245492457530770587648"
)
```

Tupper 公式把 `$k$` 编码成一个宽 106、高 17 的位图。在 $0\le x<106$、$k\le y<k+17$ 的范围内绘制满足下式的点：

$$
\frac{1}{2}<\left\lfloor\operatorname{mod}\left(\left\lfloor\frac{y}{17}\right\rfloor 2^{-17\lfloor x\rfloor-\operatorname{mod}(\lfloor y\rfloor,17)},2\right)\right\rfloor
$$

也可以直接按位恢复图像，不依赖在线解析站：

```python
from PIL import Image

# k 使用上面的完整整数
bitmap = k // 17
image = Image.new("1", (106, 17), 1)

for x in range(106):
    for y in range(17):
        if (bitmap >> (17 * x + y)) & 1:
            image.putpixel((x, 16 - y), 0)

image.resize((1060, 170)).save("tupper_flag.png")
```

生成的图像中可读出：

```text
0xGame{SSSS_dgjw}
```

## 方法总结

本题的关键证据链是“识别系统 profile → 恢复口令 `ffxiv` → 从命令行历史定位密文 → 按题面用 RC4 解密 → 将大整数解释为 Tupper 位图”。`U2FsdGVkX1` 不是算法指纹；只有把内存取证得到的口令、题面给出的算法和解密后的绘图参数串联起来，结论才可复现。
