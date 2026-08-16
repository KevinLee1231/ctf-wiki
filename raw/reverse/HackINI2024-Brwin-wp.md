# HackINI2024 Brwin

## 题目简述

服务连续生成 12 个随机的 12 位二进制数。每一关都允许提交一个最长 4096 字符的二进制字符串，只要随机目标作为某个连续 12 位子串出现就算通过。目标不是逐关猜随机数，而是构造一条尽量覆盖所有 12 位组合的通用输入。

## 解题过程

长度为 $n$、字母表大小为 $k$ 的 De Bruijn 序列 $B(k,n)$，在循环意义下恰好包含每个长度为 $n$ 的字符串一次。本题取二进制字母表和 12 位窗口：

$$
k=2,\qquad n=12,\qquad k^n=2^{12}=4096
$$

这正好等于服务允许的输入长度。可以用经典递归算法生成 $B(2,12)$：

```python
def de_bruijn(alphabet, n):
    k = len(alphabet)
    state = [0] * (k * n)
    sequence = []

    def db(t, period):
        if t > n:
            if n % period == 0:
                sequence.extend(state[1:period + 1])
            return

        state[t] = state[t - period]
        db(t + 1, period)
        for value in range(state[t - period] + 1, k):
            state[t] = value
            db(t + 1, t)

    db(1, 1)
    return "".join(alphabet[index] for index in sequence)

password = de_bruijn("01", 12)
assert len(password) == 4096
print(password)
```

连接服务后，在 12 个阶段重复提交这条序列即可。严格来说，De Bruijn 序列的全覆盖是循环性质，而服务把 4096 字符按线性字符串搜索，且缓冲区无法再追加开头 11 位，因此会缺少少量跨首尾窗口。随机目标落入这些窗口时该次连接会失败；重新连接并再次提交即可，单关成功率约为 $4085/4096$，连续 12 关的成功率仍约为 96.8%。

通过全部阶段后，实际服务附件输出：

```text
shellmates{D1D_Y0U_R3aD_4bOUt_TH3_bRU1jN_s3qu3nce?}
```

官方说明末尾保留了另一版旧 flag，但 `challenge/flag.txt` 与部署工件使用上面的值。

## 方法总结

当验证器随机选择固定长度模式，而用户只能提交一条总长度受限的字符串时，应联想到 De Bruijn 序列。它将所有 $k^n$ 个模式压缩进长度 $k^n$ 的循环序列。本题的实现没有处理首尾相接，造成小概率失败；WP 应明确这个边界，而不是宣称任意一次连接都必然通过。
