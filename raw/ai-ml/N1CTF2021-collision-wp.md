# N1CTF 2021 - collision

## 题目简述

题目把一张 $28\times 28$ 的 MNIST 灰度图送入卷积网络，并将全连接层输出的 128 个实数按正负号量化成 128 位感知哈希。目标是在尽量少改动原图的前提下，使测试图与另一张目标图得到完全相同的哈希。

仓库中的 `README` 只有 `WIP`，但官方 `exp.py` 保存了完整攻击条件：原图是测试集第 22 张图片，目标图是第 8583 张；提交必须同时满足哈希完全相同、恰好改动 54 个像素，并且均方误差小于 $0.053$。因此本题的决定性障碍是针对可微模型生成受约束对抗样本，应归入 AI/ML。

## 解题过程

### 还原哈希函数

网络依次经过两层卷积、ReLU、最大池化、Dropout 和全连接层，题目并不使用最后的分类层，而是直接取 `fc1` 的 128 维输出。每一维只保留符号：

```python
def hex_hash(features):
    bits = [str(int(v > 0)) for v in features.detach().numpy()[0]]
    return hex(int("".join(bits), 2))
```

因此不必逼近目标特征的具体数值，只要让每一维的正负号一致即可。记目标输出为 $t$，待优化输出为 $z$，则官方脚本使用

$$
s=-\operatorname{sign}(t),\qquad
L_{hash}=\sum_i\operatorname{ReLU}(s_i z_i).
$$

若某一维符号已经与目标相同，该项为 0；符号相反时则产生正损失。

### 同时约束哈希与图片改动

只优化哈希会在大量像素上留下很小的扰动，无法通过严格的 $L_0$ 条件。官方脚本把图片差异的 L1 损失与哈希损失相加：

```python
adv_out = model(adv)
image_loss = nn.L1Loss()(adv, image) * 10
hash_loss = torch.sum(F.relu(-torch.sign(target_out) * adv_out))
loss = image_loss + hash_loss
loss.backward()

adv = (adv - 0.1 * adv.grad).clamp(0, 1)
```

当哈希首次碰撞后，再把变化很小的像素强制恢复成原值。阈值随碰撞次数逐渐增加，促使优化把影响集中到更少的像素：

```python
if hex_hash(adv_out) == hex_hash(target_out):
    threshold = np.clip(0.02 + 0.04 * collision_count, 0.02, 0.4)
    mask = (torch.abs(adv - image) < threshold).float()
    adv = adv * (1 - mask) + image * mask
    collision_count += 1
```

不断重复“梯度更新—哈希检查—小扰动归零”，直到同时满足：

```python
hex_hash(adv_out) == hex_hash(target_out)
torch.sum(adv != image) == 54
nn.MSELoss()(adv, image) < 0.053
```

最后将张量转成 `float32` 原始字节并 Base64 编码，完成服务端给出的四字符 SHA-256 PoW 后提交：

```python
payload = adv.squeeze().detach().numpy().astype("float32").tobytes()
io.sendline(base64.b64encode(payload))
```

仓库没有保留模型参数、MNIST 数据和远端返回结果，因此无法在当前材料中复现最终 flag；不过上述优化链条和成功判定均直接来自官方解题脚本。外部复盘 [FGSM：从论文到实战](https://cn-sec.com/archives/1290294.html) 也给出了同一脚本，并解释了符号损失与渐进掩码的作用。

## 方法总结

本题不是寻找传统文件哈希碰撞，而是利用模型可微性控制感知哈希的符号位。关键是把目标拆成两个阶段：先用可导的符号损失得到 128 位完全碰撞，再用渐进式硬掩码把分散扰动压缩到 54 个像素。遇到同时限制 $L_0$ 与 $L_2$ 的对抗样本题时，单纯 FGSM 往往不够，还需要显式稀疏化或逐步剪枝。
