# DownUnderCTF 2022 faulty arx Writeup

## 题目简述

题目实现了一个接近 ChaCha 的 20 轮 ARX 流生成器。状态由常量 `downunderctf2022`、32 字节 ASCII 十六进制密钥、计数器和 nonce 组成。最后一个双轮的四个 diagonal quarter-round 中会随机选择一个，把本应旋转 12 位的操作改成旋转 $12\oplus1=13$ 位；也可能完全不注入故障。

同一个密钥、nonce 和 counter 被重复计算 20 次，服务输出去重后的密钥流块。因此可以同时取得正确输出和四种固定故障输出，目标是用差分故障分析恢复密钥。

## 解题过程

每种故障只影响一个 diagonal 的四个输出 word。对所有输出逐 word 统计众数，可以恢复无故障密钥流；再按与正确值不同的位置识别四种故障：

```text
[0, 5, 10, 15]  -> key words (k1, k6)
[1, 6, 11, 12]  -> key words (k2, k7)
[2, 7, 8, 13]   -> key words (k3, k4)
[3, 4, 9, 14]   -> key words (k0, k5)
```

```python
correct = [Counter(words).most_common(1)[0][0]
           for words in zip(*ciphertexts)]

def find_fault(diagonal):
    return next(ct for ct in ciphertexts
                if all(ct[i] != correct[i] for i in diagonal))
```

故障只发生在最后一轮，因此无需逆完整 20 轮。对一个 diagonal 的正确/故障输出写出最后一个 quarter-round 方程，唯一差别是 `ROL12` 与 `ROL13`。由第一个输出差分可列出旋转前中间量的三个进位候选；再枚举一个 32 位中间 word，使其同时满足另外两个输出差分方程。

从 feed-forward 关系 `output = initial_state + final_state mod 2^32` 中减去求得的最终状态，即可得到该 diagonal 涉及的两个 key word。密钥本身由 `os.urandom(16).hex().encode()` 生成，所以每个 key word 的四个字节必须都是十六进制 ASCII；这个强约束会大量过滤候选。官方实现用编译后的 C++ 循环完成 $2^{32}$ 中间量枚举。

四组 diagonal 分别得到候选对后，枚举其笛卡尔积，重新运行无故障加密并检查生成块是否出现在服务输出集合中：

```python
for parts in product(cands_16, cands_27, cands_34, cands_05):
    k1, k6 = parts[0]
    k2, k7 = parts[1]
    k3, k4 = parts[2]
    k0, k5 = parts[3]
    key = words_to_bytes([k0, k1, k2, k3, k4, k5, k6, k7])
    if faulty_arx(key, nonce).stream(64) in observed:
        break
```

提交当前连接动态生成的 32 字节密钥后得到：

```text
DUCTF{las3r_b34ms_4nd_g4mm4_r4ys_p0s3_a_s3r10us_thr34t_t0_cryp70gr4phy!1!!}
```

## 方法总结

差分故障分析的优势在于故障发生得越靠后，需要逆向的轮数越少。本题先利用多样本逐 word 众数分离正确输出，再用受影响位置定位故障 round，最后以密钥字符集约束削减候选。面对随机注入的局部故障，应优先寻找“正确/故障样本配对、故障传播范围和秘密格式约束”这三类信息，而不是尝试暴力破解整个密钥。
