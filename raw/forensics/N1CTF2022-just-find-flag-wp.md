# N1CTF 2022 - just find flag

## 题目简述

题目附件是一份 Windows 7 SP1 x64 内存镜像 `mem.raw`。目标是从内存中恢复被加密的 `flag.zip`，再根据系统中留下的线索推导 ZIP 密码。

这道题的关键不是单次文件雕刻，而是把三类证据串起来：内存中的 ZIP 数据、命令历史给出的壁纸提示，以及 MFT 中保存的原始文件路径。

## 解题过程

### 定位并雕刻 ZIP

先用 `strings` 搜索与 flag 有关的字符串：

```bash
strings mem.raw | grep -i flag
```

镜像中可以看到 `flag.zip`、`C:\Users\dora\Desktop\flag.zip` 和 `flag.txt` 等痕迹。用 `foremost` 雕刻 ZIP：

```bash
foremost mem.raw
ls output/zip
```

输出中有两个 ZIP，其中 `01377758.zip` 包含加密的 `flag.txt`。

### 从命令历史找到密码规则

镜像对应 Windows 7 SP1 x64，可以用 Volatility 2 的 `Win7SP1x64` profile。查看命令历史：

```bash
volatility -f mem.raw --profile=Win7SP1x64 cmdscan
```

其中有一条提示：

```text
Stucked? You can ask WallPaper god for help.
```

据此搜索默认壁纸相关文件：

```bash
volatility -f mem.raw --profile=Win7SP1x64 filescan |
  grep -E 'Wallpaper|img0'
```

结果中除了正常的 `img0.jpg`，还有一个可疑的 `img0.jpeg`：

```text
0x000000007fc48f20 ... \Windows\Web\Wallpaper\Windows\img0.jpeg
```

将它导出：

```bash
volatility -f mem.raw --profile=Win7SP1x64 \
  dumpfiles -Q 0x7fc48f20 -D dumped
```

导出图片只承载文字提示，没有额外视觉信息。其内容可直接转写为：ZIP 密码等于目标文件完整路径经过 MD5 后的十六进制摘要，且要使用真实路径而不是桌面快捷方式路径。

### 从 MFT 恢复真实路径

普通 `filescan` 没有直接给出目标的真实路径，继续解析 MFT：

```bash
volatility -f mem.raw --profile=Win7SP1x64 mftparser |
  grep -i flag
```

关键记录为：

```text
PROGRA~2\WINDOW~2\ACCESS~1\flag.zip
```

在该系统中，短文件名对应的两个合理候选是：

```text
C:\Program Files (x86)\Windows NT\Accessories\flag.zip
C:\Program Files\Windows NT\Accessories\flag.zip
```

分别计算 MD5，第一条路径得到正确密码。Python 中应使用原始字符串，避免反斜杠被解释为转义序列：

```python
import hashlib

path = r"C:\Program Files (x86)\Windows NT\Accessories\flag.zip"
password = hashlib.md5(path.encode()).hexdigest()
print(password)
```

输出为：

```text
0d3ba7db468bdbd4f93a88c97ba7bef1
```

用该密码解压 `01377758.zip`，读取 `flag.txt`：

```text
n1ctf{0ca175b9c0f7582931d89e2c89231599}
```

## 方法总结

内存取证题中，`strings` 和文件雕刻只能恢复数据，不一定能恢复解密所需的上下文。本题需要先用 `cmdscan` 找提示，再用 `filescan` 导出壁纸中的密码规则，最后用 `mftparser` 恢复真实路径。短文件名只是线索，必须结合系统目录结构展开并逐个验证；计算 Windows 路径哈希时还要注意 Python 的反斜杠转义。
