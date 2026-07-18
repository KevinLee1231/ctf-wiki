# deeptx

## 题目简述

程序读取一张 $256\times256$、8 位、无压缩 BMP，并通过三个 CUDA kernel 逐层变换像素。附件只给出加密程序和 `deep_flag.bmp`，需要从 CUDA fatbin 中恢复 PTX，逆转置换、修改版 TEA、矩阵乘法和逐字节替换，最后去除运动模糊得到原图。

输入共有 $256^2=65536$ 个像素。三个 kernel 都以 `<<<256, 256>>>` 启动，因此 `blockIdx.x` 可视为行坐标，`threadIdx.x` 可视为列坐标。

## 解题过程

### 提取并划分 PTX

先从宿主 ELF 中导出 PTX：

```bash
cuobjdump --dump-ptx ./quiz > quiz.ptx
```

fatbin 中包含三个入口 `_Z6Layer1PhS_`、`_Z6Layer2PhS_` 和 `_Z6Layer3PhS_`，以及常量数组：

```text
cuda_sbox[256]
cuda_tbox[256]
cuda_motion[256]   # 16 × 16 个 float
```

应同时从 fatbin 或宿主初始化代码中导出 `sbox`、`tbox` 和运动模糊核。无需在 WP 中保留整份 PTX；每层的索引式和逆运算才是有效信息。

### 逆 Layer3

Layer3 最复杂，正向顺序为：

1. 每字节异或 `block_id | thread_id`；
2. 每 8 字节执行 $0x316aa7$ 轮修改版 TEA；
3. 每字节异或 `block_id & thread_id`；
4. 每行乘以一个由 `sbox`、`tbox` 生成的 $256\times256$ 矩阵，运算模 $256$；
5. 对每个输出字节执行从 8 到 `0x3f235e` 的迭代替换。

所以逆序首先处理最后的字节迭代。正向单轮可整理为：

$$
x' = S\left(T[i\bmod256]\oplus\left(13\cdot\operatorname{ROL}_8(x,3)+(b\oplus t)\right)\right),
$$

其中 $b$、$t$ 分别是 block 和 thread 编号。因为 $13^{-1}\equiv197\pmod{256}$，逆运算为：

```cpp
for (uint32_t i = 0x3f235e; i >= 8; --i) {
    ch = inv_sbox[ch];
    ch ^= tbox[i & 0xff];
    ch -= block_id ^ thread_id;
    ch *= 197;
    ch = rol8(ch, 5);          // 等价于 ror8(ch, 3)
}
```

矩阵元素由下面的序列生成：

```text
j0 = sbox[row]
M[row][col] = tbox[j_col]
j_(col+1) = 5 * j_col + 17 mod 256
```

对 $M$ 在环 $\mathbb Z/256\mathbb Z$ 上求逆，得到 $M^{-1}$，再对每个 256 字节行向量做逆矩阵乘法。不能直接使用浮点矩阵求逆；元素和结果必须始终模 $256$，并检查：

$$
M^{-1}M\equiv I\pmod{256}.
$$

接着撤销 `block_id & thread_id` 的异或。修改版 TEA 每 8 字节一组，逆向时使用源码中的终值 `sum = 0xb6b22a99`，按相反顺序执行 $0x316aa7$ 轮：

```cpp
for (uint32_t i = 0; i < 0x316aa7; ++i) {
    sum -= 0x9a28b107;
    v0 += ((v1 << 4) + 0x250de9f3) ^
          (v1 + sum) ^ ((v1 >> 5) + 0xcc981337);
    v1 -= ((v0 << 4) + 0x52a9002c) ^
          (v0 + sum) ^ ((v0 >> 5) + 0x77a13408);
}
```

所有加减均按 `uint32_t` 自然溢出。最后再次异或 `block_id | thread_id`，完成 Layer3 还原。

### 逆 Layer2

Layer2 不改变字节值，只按 `sbox` 同时置换行列。PTX 对输入坐标 $(b,t)$ 执行：

$$
out[256\cdot S(t)+S(b)] = in[256\cdot b+t].
$$

计算逆 S 盒后可直接恢复：

```cpp
out[256 * inv_sbox[thread_id] + inv_sbox[block_id]] =
    in[256 * block_id + thread_id];
```

先检查 `inv_sbox[sbox[i]] == i`，否则一个错误的常量提取就会让整张图像看似随机。

### 逆 Layer1

Layer1 对每个有效输出位置做 $16\times16$ 卷积。`cuda_motion` 的命名和核形状表明这是运动模糊；PTX 中只有行列均小于 241 的线程执行卷积，正好对应 $256-16+1=241$ 个 valid-convolution 位置，其余边界写零。

令模糊图为 $B$、原图为 $I$、卷积核为 $K$。频域中有：

$$
\mathcal F(B)=\mathcal F(I)\,\mathcal F(K).
$$

直接逆滤波会在 $\mathcal F(K)$ 接近零的位置放大噪声，实际应使用带正则项的 Wiener 形式：

$$
\widehat I=mathcal F^{-1}\left(
\frac{\overline{\mathcal F(K)}}{|\mathcal F(K)|^2+\lambda}
\mathcal F(B)
\right).
$$

实现时把 $16\times16$ 核补零到 $256\times256$，按卷积原点做循环移位，再调节较小的 $\lambda$。由于正向只保留 valid 区域，边缘无法完全无损恢复，但 flag 位于主体区域，Wiener 反卷积后已足够辨认。

### 输出 BMP

解密过程中只替换 65536 字节像素区，原 BMP 文件头、信息头和 256 色调色板应原样写回。推荐每逆一层输出一张中间图：Layer3 后检查是否出现稳定纹理，Layer2 后检查几何排列，Layer1 后再判断文字是否可读。这样比只看最终结果更容易定位错误层。

## 方法总结

这道题应按数据流逆序拆解，而不是逐条翻译整份 PTX。Layer3 的关键是识别 8 位旋转、模 $256$ 逆元、矩阵逆和 `uint32_t` 溢出；Layer2 是双轴置换；Layer1 是非完全可逆的图像复原问题，应采用正则化反卷积。CUDA 逆向中，`blockIdx`、`threadIdx`、同步点和常量内存访问共同决定算法边界，先恢复高层索引式再写逆程序，错误率最低。
