# TSGCTF2020 Poor Stego Guy WP

## 题目简述

生成脚本先用 ImageMagick 生成 $128\times128$ 的随机灰度噪声，并以 JPEG quality 50 保存；随后 Pillow 解码该 JPEG，把 `secret.zip` 的每一位与对应像素做 XOR，最后保存为无损 PNG：

```python
pixel = image.getpixel((x, y))
image.putpixel((x, y), pixel ^ secret_bit)
```

若底图是真正均匀随机的 PNG，这相当于一次一密，仅凭输出无法恢复 secret。可解性来自 JPEG 的有损量化：解码后的每个 $8\times8$ 像素块只能落在由整数 DCT 系数和固定量化表生成的稀疏集合中，而 XOR 只让观测像素相对原值改变 0 或 1。于是恢复底图可建模为 64 维格上的最近向量问题。

## 解题过程

quality 50 使用标准亮度量化表。对每个频率 $(u,v)$，量化后的整数系数 $z_{u,v}$ 经过反量化和二维 IDCT，对像素 $(x,y)$ 的贡献为：

$$
\frac{C_uC_v}{4}q_{u,v}z_{u,v}
\cos\frac{(2x+1)u\pi}{16}
\cos\frac{(2y+1)v\pi}{16},
$$

其中 $C_0=1/\sqrt2$，其余 $C_u=1$。官方 Sage solver 为 64 个系数各建立一个 128 维基向量：前 64 维是系数单位向量，后 64 维是该系数对所有像素的 IDCT 贡献，并整体乘 $2^{32}$ 转为整数。观测块的目标向量前半为 0，后半为：

```python
(output_pixel - 128) * (2 ** 32)
```

对统一格基先做 LLL 约化，再对 256 个图像块分别运行 Babai nearest-plane：

```python
lattice = IntegerLattice(vectors, lll_reduce=True)
gram = lattice.reduced_basis.gram_schmidt()[0]

for block in blocks:
    target = vector(ZZ, [0] * 64 + scaled_pixels(block))
    nearest = Babai_closest_vector(lattice.reduced_basis, gram, target)
    quantized_coefficients.append(nearest[:64])
```

得到量化系数后，不宜直接用高精度 IDCT 计算像素，因为题目实际使用 Pillow/libjpeg 的整数舍入路径；差 1 就会破坏隐藏位。solver 按 JPEG zig-zag 顺序、DC 差分、AC 的 run-length/size 编码重新生成一个 JPEG，写入相同量化表和自定义 Huffman 表，再让 Pillow 解码 `restored.jpg`。这保证恢复像素经过与题目一致的解码舍入。

最后逐像素计算：

```python
bitstream += str(output_pixel ^ restored_pixel)
```

每 8 位转为字节得到 `secret.zip`，解压后是一张只包含文字的 flag 图片。该图片没有额外视觉机制，因此直接转写而不保留截图：

```text
TSGCTF{YoU_aR3_tHe_4b5oLuTe_w1nNer_of_JPEG!}
```

## 方法总结

表面上是 LSB 隐写，决定性障碍却是利用 JPEG 量化结构从带一位噪声的像素中恢复精确底图，因此归入 Crypto 的 lattice/CVP 方向。解法把整数 DCT 系数到像素的线性映射构造成格，用 LLL 与 Babai 找最近合法 JPEG 状态，再通过重建 JPEG，复用原解码器的舍入行为。处理有损格式时，“数学上最精确”的逆变换未必能复现实现输出；用于提取最低位时，编码器版本、量化表和整数舍入都必须一致。
