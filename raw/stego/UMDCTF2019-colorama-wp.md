# UMDCTF 2019 - Colorama

## 题目简述

附件是一张由少量高饱和色块组成的 PNG。颜色落在 Piet 语言的标准调色板上，因此图像本身是一段二维图形化程序。

## 解题过程

原图中的色块、白色通道和黑色边界都具有控制流意义，不能用 OCR 或颜色压缩替代：

![由 Piet 标准色块组成的二维程序](./UMDCTF2019-colorama-wp/piet-program.png)

图像很小，可以先放大确认每个 codel 的边界，再转换成 `npiet` 接受的无损 PPM：

```bash
convert colorama.png colorama.ppm
npiet colorama.ppm
```

解释器按照相邻色块的色相变化和明度变化执行入栈、算术与输出操作，最终打印：

```text
UMDCTF-{Arent_you_glad_we_didnt_do_piet_RE_instead?}
```

其 SHA-256 与官方摘要一致。

## 方法总结

识别 Piet 的关键特征是有限的标准颜色、成块的像素和白/黑区域。此类图像不是“藏了文字”的普通隐写载体，而是可执行程序；必须保留原始像素和视觉布局。转换格式时应使用无损方式，避免调色板变化破坏语义。
