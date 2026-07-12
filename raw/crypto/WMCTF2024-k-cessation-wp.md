# k_cessation

## 题目简述

题目实现 64-Cessation 古典密码。源码注释说明：轮子长度为 64，每个明文 bit 都让轮子向前转到下一个相同 bit，并把转动距离作为密文。flag 是 ASCII、长度大于 64，且不以 `WMCTF{` 或 `FLAG{` 开头；每个字节最高位还会随机翻转。

关键源码：

```python
def __find_next_in_wheel(self, target):
    result = 1
    while True:
        ptr = (self.state + result) % len(self.wheel)
        if self.wheel[ptr] == target:
            self.state = ptr
            return [result]
        result += 1
```

## 解题过程

### 关键机制

密文的每个距离会给轮子位置之间的不等关系。例如当前指针为 $s$，距离为 $d$，则 $(s+d)\bmod64$ 是目标 bit；中间所有位置都不是目标 bit，所以它们与目标位置取值相反。

源码还给出盐化 SHA256 校验：

```python
h = hashlib.sha256((salt + flag).encode()).hexdigest()
```

虽然每个字节最高位随机翻转，但解密后可以用：

```python
b = b & 0b01111111
```

恢复 ASCII 低 7 位。

### 求解步骤

用 z3 建立 64 个 `0/1` 变量。遍历密文距离，加入约束：

```python
wheel = [z3.Int(f"wheel_{i}") for i in range(64)]
for i in range(64):
    solver.add(z3.And(wheel[i] >= 0, wheel[i] <= 1))

wheel_ptr = -1
for distance in ct:
    position = wheel_ptr + distance
    position_real = position % 64
    for i in range(position - 1, wheel_ptr, -1):
        solver.add(wheel[position_real] != wheel[i % 64])
    wheel_ptr = position_real
```

枚举 z3 模型，用候选 wheel 解密密文，清掉每字节最高位，然后用 `flag_hash.txt` 校验：

```python
def try_decrypt(key):
    h = K_Cessation(key)
    flag = decode_ascii_with_random_msb(h.decrypt(ct))
    return hashlib.sha256(salt.encode() + flag).hexdigest() == plaintext_hash
```

解出原文：

```text
DoubleUmCtF[S33K1NG_tru7h-7h3_w1s3-f1nd_1n57e4d-17s_pr0f0und-4b5ence_n0w-g0_s0lv3-th3_3y3s-1n_N0ita]
```

按题干要求改成：

```text
WMCTF{S33K1NG_tru7h-7h3_w1s3-f1nd_1n57e4d-17s_pr0f0und-4b5ence_n0w-g0_s0lv3-th3_3y3s-1n_N0ita}
```

## 方法总结

- 距离密文不是直接泄露 bit，而是泄露“目标位置与中间位置取反”的约束。
- 最高位随机翻转只影响 ASCII 的第 8 bit，低 7 bit 可保留。
- 盐化 hash 用于从多个 wheel 候选中确认唯一明文。
