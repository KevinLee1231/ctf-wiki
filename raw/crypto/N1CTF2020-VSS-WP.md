# N1CTF 2020 VSS Writeup

## 题目简述

题目实现 $2$-out-of-$2$ 视觉秘密共享：flag 先编码为 QR 码，每个像素再扩展成 $2\times2$ 黑白块，并用 Python `random.getrandbits()` 生成的伪随机比特决定两份 share 的排列。附件只给 `share2.png`。如果随机比特真正不可预测，单份 share 不泄露秘密；但 Python `random` 使用 MT19937，只需连续 $624\times32$ 位输出即可恢复内部状态。

## 解题过程

### 从 share2 还原每个随机比特

每个原像素对应的 $2\times2$ 块中，左列上下像素相同，右列上下像素相同。读取左上像素最低位即可得到“原像素与随机翻转位异或后的值”：

```python
for index in range(width * height):
    row, col = divmod(index, width)
    left = image.getpixel((2 * col, 2 * row)) & 1
    observed.append(left)
```

QR 码四周有固定白色静区。题目生成尺寸为 $444\times444$，末尾足够长的一段像素必为白色，因此这部分原像素已知。把已知白色位与 `observed` 异或，就得到连续的 MT19937 输出位。

### 恢复 MT19937 状态

取最后 $624\times32$ 位，按生成器输出的位序重新组成 32 位整数，并对每个输出执行 untemper，恢复 624 个状态字。也可使用 `MT19937Predictor`/`RandCrack` 逐块提交：

```python
known = original_white_bits ^ observed_tail
predictor.setrandbits(bits_as_integer, 624 * 32)
```

位序是本题最容易出错的地方。应先用题目源码生成一张小测试图，确认 `bin(random.getrandbits(n))[2:].zfill(n)` 与图像扫描顺序一致，再处理附件。

### 预测其余掩码并重建 QR

状态恢复后，预测静区之前的所有随机位，并与已经取得的尾部位拼接成完整 `flip` 序列：

```python
plain = [seen ^ mask for seen, mask in zip(observed, flip)]
```

把 0/1 映射回黑白像素，按原始宽高保存为 QR 图，再用任意 QR 解码器读取 flag。这里不需要保留解出的二维码截图：二维码只是最终文本的机器可读表示，关键证据是 MT 状态恢复和像素重建关系。

## 方法总结

视觉秘密共享的安全性依赖掩码不可预测，而不是图片看起来像噪声。MT19937 适合模拟，不适合生成密码学掩码；一旦输出中存在已知明文区域，就可恢复整个状态。处理图像密码题时，应把像素遍历顺序、颜色到比特的映射和 PRNG 输出位序分别验证，避免“算法正确但图像仍乱码”的实现错误。
