# Diffecient

## 题目简述

题目用 Bloom Filter 保存随机生成的 256 字节密钥。管理员验证要求输入至少 32 字节，并同时含有小写字母、大写字母、数字和特殊字符；只要该输入在 Bloom Filter 中命中，服务就返回 flag。

普通做法只能等待误报，但服务还允许免费加入一个样本。关键在于 Bloom Filter 的 47 个位置全部由 `mmh3.hash(key, seed)` 计算，而 MurmurHash3 存在可以对所有种子同时成立的等长碰撞。先加入一个不满足管理员策略的碰撞串，再用满足策略的另一串验证，就能绕过 `added_keys` 检查。

## 解题过程

Bloom Filter 的判断逻辑为：

```python
def check(self, item):
    return all(
        self.hash(item, i) % self.__m in self.__digests
        for i in range(self.__k)
    )
```

而管理员路径额外拒绝刚刚由用户加入的原文：

```python
if key in self.added_keys:
    return False
return self.check(key)
```

因此需要构造两个不同的字节串 $a\ne b$，使 47 个种子下都有：

$$
\operatorname{mmh3}(a,i)=\operatorname{mmh3}(b,i),
\qquad 0\le i<47.
$$

MurmurHash3 x86-32 会先对每个 4 字节块做可逆变换：

```python
def forward_block(block):
    n = int.from_bytes(block, "little")
    n = n * 0xcc9e2d51 & 0xffffffff
    n = (n >> 17) | (n << 15) & 0xffffffff
    n = n * 0x1b873593 & 0xffffffff
    return n.to_bytes(4, "little")
```

官方脚本使用固定差分：

```python
DIFF = b"\x00\x00\x04\x00\x00\x00\x00\x80"

def collision8(text):
    changed = bytes(
        x ^ y
        for x, y in zip(DIFF, forward(text))
    )
    return invert(changed)
```

这个差分作用在连续两个内部块上，使两条 MurmurHash3 状态在处理完第二块后重新汇合；后续块、长度混合和最终化简都相同，所以碰撞与初始种子无关。

选择满足管理员复杂度要求的 32 字节字符串：

```python
valid = b"A" * 28 + b"Aa1!"
colliding = collision8(valid[:8]) + valid[8:]
```

本地逐种子验证可得到：

```text
valid     = 4141414141414141414141414141414141414141414141414141414141613121
colliding = e99f204c4141f27c414141414141414141414141414141414141414141613121
all 47 hashes equal: True
```

交互顺序如下：

1. 选择添加样本，把 `colliding` 的十六进制提交到 Bloom Filter；
2. 选择管理员验证，提交 `valid`；
3. `valid` 不在 `added_keys` 中，却与已加入的 `colliding` 设置完全相同的 47 个位置，因此验证通过。

最终得到：

```text
SEKAI{56f066a1b13fd350ac4a4889efe22cb1825651843e9d0ccae0f87844d1d65190}
```

## 方法总结

Bloom Filter 只能表示“可能存在”，不能证明元素确实被录入；把它直接当作身份验证数据库本就会受到误报影响。本题进一步利用非密码哈希 MurmurHash3 的种子无关碰撞，使攻击者可以主动制造误报。安全认证需要保存并比较不可伪造的认证数据，不能只依赖 Bloom Filter，也不应把快速非密码哈希当作抗碰撞原语。
