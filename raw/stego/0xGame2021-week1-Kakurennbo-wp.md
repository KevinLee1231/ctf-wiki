# week1Kakurennbo

## 题目简述

附件是一段表面正常的文本，实际在可见字符之间插入了 Unicode 不可见字符。第一层按四种不可见字符编码隐藏文本，解出后得到一段 21 栏 W 型栅栏密码，继续还原即可得到 flag。

## 解题过程

在文本中逐字符移动光标时会出现不自然的停顿，用 Vim 的可见字符模式或十六进制编辑器检查可确认其中混有不可见 Unicode 码点。原题所用编码器的默认字符表如下：

- `U+200C` 表示四进制数字 `0`；
- `U+200D` 表示四进制数字 `1`；
- `U+202C` 表示四进制数字 `2`；
- `U+FEFF` 表示四进制数字 `3`。

每 8 个四进制数字表示一个 UTF-16 码元，因为 $4^8=65536$。下面的代码可以直接从附件中提取第一层隐藏文本：

```python
from pathlib import Path

mapping = {
    "\u200c": "0",
    "\u200d": "1",
    "\u202c": "2",
    "\ufeff": "3",
}

text = Path("attachment.txt").read_text(encoding="utf-8")
digits = "".join(mapping[ch] for ch in text if ch in mapping)
assert len(digits) % 8 == 0

hidden = "".join(
    chr(int(digits[i:i + 8], 4))
    for i in range(0, len(digits), 8)
)
print(hidden)
```

输出仍不是明文，而是 21 栏 W 型栅栏密码。其轨迹周期为 $2\times(21-1)=40$：先按 `0,1,...,20,19,...,1` 生成每个位置所属的栏，按各栏出现次数切分密文，再沿同一轨迹依次取字符。可用下面的代码解密：

```python
def rail_fence_decrypt(ciphertext, rails):
    pattern = list(range(rails)) + list(range(rails - 2, 0, -1))
    route = [pattern[i % len(pattern)] for i in range(len(ciphertext))]

    buckets = []
    offset = 0
    for rail in range(rails):
        count = route.count(rail)
        buckets.append(iter(ciphertext[offset:offset + count]))
        offset += count

    return "".join(next(buckets[rail]) for rail in route)

print(rail_fence_decrypt(hidden, 21))
```

输出即为 flag。

## 方法总结

本题的识别线索是“可见文本正常，但光标移动和字符数量异常”。解题链路为“提取四种不可见字符 → 按四进制每 8 位恢复文本 → 使用 21 栏 W 型栅栏解密”。正文已经保留原在线解码器的字符映射和算法，因此无需依赖外部网站。
