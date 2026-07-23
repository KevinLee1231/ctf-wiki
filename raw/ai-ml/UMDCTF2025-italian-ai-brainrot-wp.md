# UMDCTF 2025 - italian-ai-brainrot

## 题目简述

服务接收一组字节，直接作为 PyTorch CPU `Generator` 的状态。随后用 Stable Diffusion v1.4 的 UNet、VAE、CLIP `openai/clip-vit-large-patch14` 和 50 步 DDIM，从一个 $1\times4\times12\times12$ 的高斯潜变量生成 $96\times96$ 图像。若生成图与目标图的均方误差不超过 0.05，就返回 flag。

![服务的 96×96 目标图：一条穿蓝鞋的鲨鱼，生成结果将与其计算 MSE](UMDCTF2025-italian-ai-brainrot-wp/target-shark.jpg)

直接搜索 19968 位 MT19937 状态不可行。官方解法先对目标图做 DDIM inversion，近似恢复初始高斯噪声，再逆转正态采样过程取得 MT19937 输出的部分比特，最后在 $\mathbb F_2$ 上求一个一致的生成器状态。

## 解题过程

### 反演扩散过程

生成端使用空 prompt、guidance scale 7、50 步 DDIM，并以固定公式从时间 $t$ 更新到前一时间步。解题脚本使用完全相同的 tokenizer、CLIP、UNet、VAE、scheduler 参数，但反向遍历时间步，将目标图编码后的 latent 逐步推回噪声端。

DDIM 是确定性更新，因此在模型和调度器参数一致时可以近似反演。VAE 编解码与图像量化会产生误差，但服务只要求 MSE 小于 0.05，不要求逐像素一致。

### 从高斯样本恢复 PRNG 输出

PyTorch CPU 正态采样由 MT19937 的均匀输出经成对变换得到高斯数。官方脚本按每组 16 个高斯数配对前 8 个与后 8 个，并逆算：

$$
u=1-\exp\left(-\frac{x^2+y^2}{2}\right),
\qquad
v=\frac{\operatorname{atan2}(y,x)\bmod2\pi}{2\pi}.
$$

由于计算使用 `float32`，将 $u,v$ 乘 $2^{24}$ 并取整，可以稳定得到相应 32 位 MT 输出中可观测的 24 位，而低 8 位无需精确恢复。

### 在线性系统中构造一致状态

MT19937 的 twist 和 temper 都是 $\mathbb F_2$ 上的线性运算。`randcrack.sage` 用 624 个 32 位符号状态位模拟生成器，把每次输出的 24 个已知位写成线性方程：

$$
M\cdot s=b\pmod2.
$$

方程不足以唯一确定全部 19968 位状态，但服务只会消费当前 latent 所需的输出；求得任意一个与这些可见高位一致的解即可。把 624 个状态字按 PyTorch CPU generator 的序列化格式写成小端字节，并补上 `left`、`seeded`、`next` 和正态缓存字段：

```python
state = M.solve_right(vector(GF(2), observed_bits))
words = [
    int("".join(map(str, state[i*32:(i+1)*32])), 2)
    for i in range(624)
]
```

将完整状态字节列表提交给服务，生成图会落入目标 MSE 阈值，得到：

```text
UMDCTF{italian_ai_brainrot_is_so_goated_also_isnt_ddim_inversion_cool}
```

## 方法总结

- 核心技巧：DDIM inversion 恢复初始高斯 latent，逆 Box–Muller 型采样得到 MT19937 部分输出，再解 $\mathbb F_2$ 线性系统伪造 PyTorch RNG 状态。
- 识别信号：确定性扩散调度、攻击者可控 generator state、宽松图像相似度阈值，以及非密码学 PRNG 驱动模型输入。
- 复用要点：模型、VAE、scheduler、步数、prompt 和 guidance 参数必须完全对齐；不必执着于唯一 PRNG 状态，只要构造的状态能复现服务实际消费的随机输出即可。
