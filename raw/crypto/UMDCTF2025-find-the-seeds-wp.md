# UMDCTF 2025 - find the seeds

## 题目简述

题目给出一段 34 字节密文。生成脚本以 `int(time.time())` 作为 Python `random` 的种子，再逐字节调用 `getrandbits(8)` 生成密钥流，与 flag 异或：

```python
seed = int(time.time())
random.seed(seed)
keystream = bytes(random.getrandbits(8) for _ in plaintext)
ciphertext = bytes(p ^ k for p, k in zip(plaintext, keystream))
```

Unix 时间只有秒级熵，并且原始文件时间可以把搜索窗口缩得很小。已知 flag 前缀 `UMDCTF{` 可作为确定的种子验证 oracle。

## 解题过程

官方恢复脚本使用的生成时刻为本地时间 `2025-04-23 18:35:10`；在美国东部夏令时下等价于 `2025-04-23 22:35:10 UTC`。实际整理旧附件时，解压、复制或 Git 导出可能改写文件时间，因此应优先保留原始分发文件的时间元数据；若只有近似时刻，则在其前后枚举一个小窗口。

对每个候选秒级时间戳，重新初始化同版本 Python PRNG，只生成前 7 个字节并检查：

```python
known = b"UMDCTF{"

for seed in range(approx_time - 300, approx_time + 301):
    random.seed(seed)
    ks = bytes(random.getrandbits(8) for _ in known)
    if bytes(c ^ k for c, k in zip(cipher[:7], ks)) == known:
        break
```

找到种子后必须重新 `random.seed(seed)`，从状态起点生成与密文等长的完整密钥流；不能接着使用已经消费了 7 次输出的状态：

```python
random.seed(seed)
ks = bytes(random.getrandbits(8) for _ in cipher)
plain = bytes(c ^ k for c, k in zip(cipher, ks))
```

恢复结果为：

```text
UMDCTF{pseudo_entropy_hidden_seed}
```

## 方法总结

- 核心技巧：利用时间戳种子的低熵和已知 flag 前缀验证 Python PRNG 候选状态。
- 识别信号：`random.seed(int(time.time()))`、短时间窗口、异或流加密以及固定明文格式。
- 复用要点：记录时区和原始元数据；验证前缀后要重置 PRNG 再解完整密文。Python `random` 不是密码学安全随机数，不能直接生成密钥流。
