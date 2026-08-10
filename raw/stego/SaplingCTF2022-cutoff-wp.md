# Cutoff

## 题目简述

附件 message 没有扩展名，但文件头 89 50 4E 47 0D 0A 1A 0A 表明它是 PNG。图片只显示到靠近底部的位置，flag 被截断。问题不在 IDAT 数据丢失，而是 IHDR 中声明的高度小于实际像素行数。

## 解题过程

PNG 的 IHDR 从偏移 16 开始保存 4 字节大端宽度和高度。附件中宽度为 800，高度写成 1262。把全部 IDAT 拼接并解压后得到 3361400 字节；图片为 RGB，每行还有 1 字节滤波器，因此每行长度为：

$$
1+800\times3=2401.
$$

实际行数严格为：

$$
\frac{3361400}{2401}=1400.
$$

把高度改为 1400 后，还要重新计算 IHDR 的 CRC。修复逻辑如下：

~~~python
import struct
import zlib

data = bytearray(open("message", "rb").read())
data[20:24] = struct.pack(">I", 1400)
data[29:33] = struct.pack(
    ">I",
    zlib.crc32(data[12:29]) & 0xffffffff,
)
open("recovered-full-message.png", "wb").write(data)
~~~

修复后的完整图片右下角出现竖排文字：

![修复 IHDR 高度后，红色 800×1400 图片右下角完整显示竖排 flag](SaplingCTF2022-cutoff-wp/recovered-full-message.png)

将这段文字旋转阅读即可得到：

~~~text
maple{very_l0NG_PnG}
~~~

## 方法总结

修复损坏图片时不应只反复猜尺寸。PNG 可从颜色类型、位深、宽度和解压后扫描线长度推导真实高度；修改关键 chunk 后还必须同步更新 CRC。文件没有扩展名时，先依据魔数识别格式，再按格式结构验证字段。
