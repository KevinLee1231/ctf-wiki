# L3akCTF 2025 face_id Writeup

## 题目简述

题目提供一个“人脸登录”程序和已注册用户的数据文件 `user.db`。程序接收一张 ASCII P3 PPM 图片，只支持 $500\times500$ 尺寸；如果图片特征与数据库记录足够接近，就会输出 flag。

程序实际保存的不是人脸特征向量，而是二值图像每一行、每一列的计数。只要构造任意具有相同行列和的矩阵即可通过认证，因此决定性障碍是逆向特征提取和伪造输入，本文按 Reverse 归档。

## 解题过程

### 还原特征与数据库格式

程序把每个像素的三个通道取平均得到亮度，并从图像背景区域统计第一、第三四分位数。设四分位距为 $IQR=Q_3-Q_1$，程序使用约 $1.5IQR$ 的上下界，将位于亮度范围内的像素记为 1，其余记为 0。

接着它只保留二值矩阵的 500 个行和与 500 个列和。`user.db` 采用小端序，布局为：

```text
uint64  vertical_count = 500
int32   vertical_sums[500]
uint64  horizontal_count = 500
int32   horizontal_sums[500]
int32   lax
```

附件中的 `lax` 为 20，但没有必要依赖误差范围，因为可以直接构造精确满足两组计数的矩阵。读取数据库的核心代码如下：

```python
import struct

with open("user.db", "rb") as f:
    nv = struct.unpack("<Q", f.read(8))[0]
    vertical = list(struct.unpack(f"<{nv}i", f.read(4 * nv)))
    nh = struct.unpack("<Q", f.read(8))[0]
    horizontal = list(struct.unpack(f"<{nh}i", f.read(4 * nh)))
    lax = struct.unpack("<i", f.read(4))[0]
```

两组计数的总和都为 88162，说明存在相应的二值矩阵。

### 按行列度数构造矩阵

把每一行看作二分图左侧顶点、每一列看作右侧顶点，目标行列和就是各顶点的度数。对每一行，从剩余容量最大的列中选择所需数量的列置 1，并相应减少列容量：

```python
def build_matrix(row_sums, col_sums):
    n = len(row_sums)
    remain = col_sums[:]
    matrix = [[0] * n for _ in range(n)]

    rows = sorted(range(n), key=lambda i: -row_sums[i])
    for i in rows:
        cols = sorted(range(n), key=lambda j: -remain[j])
        for j in cols[:row_sums[i]]:
            matrix[i][j] = 1
            remain[j] -= 1

    assert all(v == 0 for v in remain)
    return matrix
```

官方脚本使用等价的贪心构造。输出 PPM 时，将 1 写成灰度 128、0 写成灰度 0。这样由大量 0 像素确定的四分位范围会让 128 像素稳定落入程序采用的分类一侧，从而复现目标二值矩阵：

```python
with open("attack.ppm", "w", newline="\n") as out:
    out.write("P3\n500 500\n")
    for row in matrix:
        out.write(" ".join(
            f"{128 if bit else 0} {128 if bit else 0} {128 if bit else 0}"
            for bit in row
        ) + "\n")
```

这里必须输出 LF 换行。题目解析器只删除 `\n`，不会清除 Windows 文本模式产生的 `\r`；直接生成 CRLF 文件会导致像素数字解析失败。

把 `attack.ppm` 交给程序后，两组投影均通过检查，得到：

```text
L3AK{mayb_image_brightness_verification_isnt_enough_hopefully_my_second_version_is_securerer}
```

## 方法总结

本题的“人脸识别”丢弃了像素位置关系，只保留行和与列和。大量不同图像会映射到同一特征，认证系统因而只是在验证一个二分图度数序列。泄露的模板数据足以直接合成碰撞输入，不需要恢复原始人脸。

处理自定义文本图片格式时，除了算法还要核对解析细节。尺寸、通道范围和换行符都会影响程序是否真正读到预期矩阵；本题中 CRLF 残留就是一个与核心构造无关、但会阻断复现的实现陷阱。
