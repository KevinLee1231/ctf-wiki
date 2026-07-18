# week3easyMisc

## 题目简述

题目入口是一个下载按钮损坏的网页，仓库中同时提供 `index.html`、`hint.png`、`record.wav` 和加密的 `flag.zip`。解题链由网页注释、PNG LSB 隐写、DTMF 音频和加密 ZIP 串联而成。

仓库里的 HTML 写的是 `hint.jpg`，实际附件名却是 `hint.png`；这是打包时的文件名不一致，不影响对真实 PNG 的分析。

## 解题过程

查看 HTML 源码可以找到两段 Base64 注释：

```text
ZWFzeU1pc2MvZmxhZy56aXA=        -> easyMisc/flag.zip
ZWFzeU1pc2MvcmVjb3JkLndhdg==    -> easyMisc/record.wav
```

先对 `hint.png` 使用 `stegpy`。该工具会按自身头部格式从图像颜色分量的低有效位重组载荷；这里不是凭肉眼观察位平面，而是让对应编码器解析其 `stegv3` 头、长度和正文。

```text
$ stegpy hint.png
get_the_password_from_wav
```

提示要求从 `record.wav` 中取得密码。音频由一段段双音多频信号组成：DTMF 每个按键同时使用一个低频组频率（697、770、852、941 Hz）和一个高频组频率（1209、1336、1477、1633 Hz），两者的行列组合对应电话键盘字符。将 WAV 按静音切段，分别取每段最强的低、高频峰并查表，得到：

```text
2821876761
```

用该十位数字解压附件：

```text
unzip -P 2821876761 flag.zip
```

经仓库附件核验，ZIP 中只有一个成员 `flag.png`，不存在原 WP 所写的“冷静分析.png”。继续运行：

```text
$ stegpy flag.png
[旧赛事域名]/0xgame/easyMisc/511a1a36eb4da797618c998ae933d72a.php
```

第二张图隐藏的是原比赛的最终 flag 页面地址，而不是 flag 文本本身。该临时域名现已返回 502，公开网页存档也无法恢复页面正文；源码仓库没有收录服务器端页面。因此正文保留它在解题链中的作用和唯一的路径指纹，不把失效域名作为需要访问的外链。

`stegpy` 是本题实际使用的 LSB 载荷格式解析器；其官方仓库同时说明了直接以图片文件为参数的提取方式：[izcoser/stegpy](https://github.com/izcoser/stegpy)。即使不打开该链接，上文也已经给出了本题所需的工具机制、命令与两次提取结果。

## 方法总结

完整链路是“HTML 注释定位附件 → stegpy 解出音频提示 → DTMF 还原十位密码 → 解压 `flag.zip` → stegpy 解出最终页面路径”。外部临时页面只承担比赛期间的最终展示功能，失效后没有复用价值；真正可复现的证据是仓库附件、密码和 PNG 中的隐藏载荷。
