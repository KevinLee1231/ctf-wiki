# CakeCTF2021 rflag

## 题目简述

服务随机生成 32 个十六进制字符，只允许进行 4 轮查询。每轮输入一个 Rust 正则表达式，服务返回该正则在秘密字符串中的全部匹配起始位置；四轮后必须提交完整字符串。

每个十六进制字符只有 4 位信息，而一次单字符集合正则能够同时询问所有 32 个位置的某一位，因此四轮刚好足够。

## 解题过程

### 把正则响应当作并行位查询

按十六进制值的四个比特设计四个字符集合：

```text
[8-9a-f]    # bit 3
[4-7c-f]    # bit 2
[2367abef]  # bit 1
[13579bdf]  # bit 0
```

每个正则都只匹配一个字符，所以 `find_iter` 返回的起始位置正好是该比特为 1 的字符下标，不会产生跨字符或重叠匹配问题。

### 根据四轮位置集合重建答案

```python
queries = [
    "[8-9a-f]",
    "[4-7c-f]",
    "[2367abef]",
    "[13579bdf]",
]
weights = [8, 4, 2, 1]
value = [0] * 32

for regex, weight in zip(queries, weights):
    sendline(regex)
    positions = parse_response()  # 例如 [0, 3, 8, ...]
    for pos in positions:
        value[pos] |= weight

answer = "".join(format(v, "x") for v in value)
sendline(answer)
```

四个响应集合共同给出每个位置的完整半字节值。提交随机答案后服务读取固定 flag：

```text
CakeCTF{n0b0dy_w4nt5_2_r3v3r53_RUST_pr0gr4m}
```

仓库官方脚本使用四个逐轮二分的集合实现同一信息提取；按独立比特设计集合更容易验证，也不需要维护区间状态。

## 方法总结

- 返回“所有匹配位置”的正则 oracle 等价于一次对整条秘密并行提问，不能只按四次普通猜测估算信息量。
- 十六进制字符恰有 4 位，四个按位字符类能无歧义覆盖整个候选空间。
- 构造 oracle 查询时应让每次匹配固定宽度为 1，避免 Rust `find_iter` 的非重叠语义引入额外状态。
