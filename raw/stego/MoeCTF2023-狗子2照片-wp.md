# 狗子(2) 照片

## 题目简述

题面明确提示 PNG 像素被修改，隐藏方式是 LSB，但“某个常用工具”无法按正确格式提取。真正的考点不是发现最低位异常，而是确认嵌入程序使用的通道顺序和每通道位数。

## 解题过程

图片是 RGBA。常见图形工具不一定按文件的原生 band 顺序导出最低位，例如某些流程会优先处理 Alpha，得到 ARGB 位流；而本题使用的 [ragibson/Steganography](https://github.com/ragibson/Steganography) 通过 Pillow 读取像素并按图像 `getbands()` 的原生顺序展平。对 RGBA PNG 而言，位流顺序就是逐像素的 R、G、B、A。

另一个容易忽略的差异是位数。该工具的 `--lsb-count` 默认值为 2，而本题只在每个通道的最低 1 位嵌入数据，所以必须显式指定 `-n 1`：

```bash
stegolsb steglsb -r -n 1 -i bincat_hacked.png -o recovered.txt
```

恢复文件中直接得到：

```text
moectf{D0ggy_H1dd3n_1n_Pho7o_With_LSB!}
```

也可以在支持自定义 LSB 配方的工具中设置等价参数：通道顺序 `R,G,B,A`，像素遍历顺序为逐行，提取每通道最低 1 位。这里真正决定结果的是这三个参数，而不是工具名称。

若使用 `zsteg` 扫描大图时出现 `stack level too deep (SystemStackError)`，这是项目中有记录的[同类问题](https://github.com/zed-0xff/zsteg/issues/30)，不能据此判断图片没有载荷。可改用上述确定参数的提取器；裁剪只保留含有前部载荷的区域也可能规避递归深度问题，但会破坏原始证据，应该对副本操作。

## 方法总结

LSB 提取失败时，应依次核对颜色通道顺序、像素遍历顺序、每通道位数和位拼接方向。题目故意让“RGBA + 1 LSB”与常用工具默认值错位，说明自动扫描结果只能作为线索，嵌入实现本身才是可靠的格式定义。
