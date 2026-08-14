# hELLO KItty

## 题目简述

附件是一张能够正常显示的 JPEG 图片，题面提示秘密隐藏在图像中。目标是识别常见的 JPEG 隐写载体并提取嵌入文件。

## 解题过程

图像表面没有需要人工辨认的异常，官方解法使用 `steghide`。先查看是否存在嵌入数据，再在无密码情况下提取：

```bash
steghide info hellokitty.jpg
steghide extract -sf hellokitty.jpg -xf answer.txt
```

若程序询问密码，直接提交空密码。提取出的 `answer.txt` 内容末尾给出：

```text
grey{easy_$T3g0_wIth_K1TTy}
```

原图只充当 `steghide` 的载体，画面本身不承载定位、像素排列或其他必要视觉线索，因此无需在 WP 中重复保存图片。

## 方法总结

能正常打开的媒体文件仍可能附带隐写数据。JPEG 题在题面明确提示 stego 工具时，可优先检查 `steghide`；真正的证据是提取命令与得到的嵌入文件，而不是普通封面画面。
