# Secure Image Encryption

## 题目简述

服务为一次请求中的所有图片选择同一个 32 位随机种子，然后用 Python `random.shuffle` 打乱像素。用户可上传两张不超过 $256\times256$ 的 PNG，服务会同时返回这两张图和 flag 图片的加密结果。

这是典型的置换密码已知明文攻击：设计两张探针图，让每个像素位置对应唯一的灰度值二元组。比较探针明文和密文中的二元组即可恢复完整置换，再把同一置换逆用到 flag 密文。

## 解题过程

加密函数先按坐标收集像素，再打乱列表：

```python
random.seed(key)
random.shuffle(pixels)
random.shuffle(pixels[: random.randint(2, len(pixels))])
```

第二次 `shuffle` 的参数是列表切片，它只打乱临时副本，不会修改原列表。因此真正生效的只有第一次置换，而且同一次请求中的三张图片都用相同种子重新初始化，置换完全一致。

一张灰度图每个像素只能保存 0 至 255。$256\times256=65536=256^2$，所以用两个灰度通道就能唯一编码所有位置：第一张图保存位置编号的低字节，第二张保存高字节。

![第一张探针图以横向灰度渐变编码位置编号的低字节](SekaiCTF2022-Secure-Image-Encryption-wp/position-low-byte.png)

![第二张探针图以纵向灰度渐变编码位置编号的高字节](SekaiCTF2022-Secure-Image-Encryption-wp/position-high-byte.png)

构造逻辑可简化为：

```python
count = 256 * 256
probe_low = [index & 0xff for index in range(count)]
probe_high = [index >> 8 for index in range(count)]
```

提交两张探针图后，对每个密文像素位置 $j$ 读取：

$$
(\text{encLow}[j],\text{encHigh}[j]).
$$

这个二元组正好是某个明文位置 $i$ 的唯一编号，因此建立：

$$
\pi(j)=i.
$$

随后按同一映射读取 flag 密文：

```python
for plain_pos in range(width * height):
    low = probe_low[plain_pos]
    high = probe_high[plain_pos]

    candidates = set(
        np.where(enc_low == low)[0]
    )
    candidates &= set(
        np.where(enc_high == high)[0]
    )

    recovered[plain_pos] = encrypted_flag[
        next(iter(candidates))
    ]
```

置换后的 flag 看不出原始结构：

![像素被统一随机置换后的 flag 密文呈现为彩色噪声](SekaiCTF2022-Secure-Image-Encryption-wp/encrypted-flag.png)

逆置换后恢复出完整图片：

![逆置换恢复出的灰度插画及对角线 flag 文本](SekaiCTF2022-Secure-Image-Encryption-wp/recovered-flag.png)

图中文字为：

```text
SEKAI{Permutation_is_not_safe_2783f169@u}
```

## 方法总结

只打乱位置、不改变像素值的图像加密无法抵抗已知明文或选择明文攻击。只要多张图片复用同一置换，就能用少量探针为每个位置编码并恢复映射。本题还包含一个常见 Python 陷阱：对切片调用原地函数只会修改副本，不能为原列表增加第二层随机化。
