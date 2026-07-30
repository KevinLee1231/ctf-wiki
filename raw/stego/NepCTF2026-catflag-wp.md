# NepCTF2026 CatFlag Writeup

## 题目简述

附件 `flag.txt` 并不是普通字符画，而是一段 Sixel 终端图像控制序列。支持 Sixel 的终端会直接把内容渲染成图片；不支持的终端则只显示大量看似乱码的转义字符。

## 解题过程

在支持 Sixel 的 Linux/Unix 终端中直接执行：

```bash
cat flag.txt
```

Windows 终端可使用：

```powershell
Get-Content -Raw -LiteralPath "flag.txt"
```

前提是当前终端实现了 Sixel 渲染。如果终端不支持，应将原始内容交给 Sixel 解码器或 `img2sixel` 配套工具转换为普通位图；无需手写几百行解析器。识别依据是文件中包含设备控制字符串及 Sixel 引导序列，随后出现调色板和像素字符。

渲染后图片中的文字为：

```text
NepCTF{Lets_Enjoy_NepCTF2026!Have_Fun!}
```

## 方法总结

本题考查终端图形格式识别。看到文本中密集的 ESC 控制序列时，应先判断它是 ANSI、Sixel、Kitty graphics 还是其他终端协议，而不是立即按编码题处理。flag 的视觉内容已经完整转写为文本，因此归档时没有必要再保留一张仅展示相同文字的截图。
