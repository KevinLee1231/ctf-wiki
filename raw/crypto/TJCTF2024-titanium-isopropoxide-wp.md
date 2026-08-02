# titanium-isopropoxide

## 题目简述

题目提供两份同一 flag 在不同随机密钥下生成的密文，以及一个自定义 RC4 风格流密码。每次加密都会从 `urandom` 取得新密钥，因此不能直接利用密钥复用；真正的问题在输出函数：状态变量初始为 `i=j=0`，更新式让 `i` 永远保持 0，输出固定来自

```text
S[S[0] & ~S[j]]
```

而不是均匀的 RC4 密钥流。该索引结构使输出字节存在明显统计偏差。

## 解题过程

先按加密代码模拟大量随机密钥。官方探索脚本生成约一百万条密钥流，统计每个位置以及全局的输出字节频率。因为两份密文是相同明文与两条独立偏置流的异或，对每个位置都可以将高频密钥流字节与密文字节异或，得到明文字符候选。

flag 字符集可限制为小写字母、下划线和花括号。对第 $i$ 位密文字节 $c_i$ 与候选明文字节 $p$，其分数取对应密钥流字节 $c_i\oplus p$ 的经验频率。两份备份的对数分数相加，可显著提高排序稳定性：

```python
import math

ALPHABET = b"abcdefghijklmnopqrstuvwxyz_{}"

def rank_char(ciphers, position, stream_frequency):
    ranked = []
    for plain in ALPHABET:
        score = 0.0
        for cipher in ciphers:
            key_byte = cipher[position] ^ plain
            probability = stream_frequency[position].get(key_byte, 1e-12)
            score += math.log(probability)
        ranked.append((score, plain))
    return sorted(ranked, reverse=True)
```

仅按单字节最高分仍可能出现局部歧义。官方 `alipo.py` 再结合 10000 词英语词表做 beam/优先队列搜索，从已知 `tjctf{` 开始扩展符合下划线分词和英语短语的高分前缀。两份仓库内明文备份也确认最终结果一致：

```text
tjctf{this_code_is_very_bad_because_you_can_see_penguins_through_it}
```

## 方法总结

- 不同密钥并不意味着多密文统计攻击失效；只要各条独立密钥流服从同一强偏置分布，同明文样本仍能联合提高置信度。
- 应先从状态更新证明偏差来源：这里 `i` 永远为 0，索引又包含按位与和取反，远非均匀置换抽样。
- 频率模型负责缩小字符候选，语言模型负责解决局部歧义。正文中的英语结构只能作为排序约束，最终结果仍应回到全部密文和加密实现进行验证。
