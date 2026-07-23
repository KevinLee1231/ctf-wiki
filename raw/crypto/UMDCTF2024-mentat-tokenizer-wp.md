# mentat tokenizer

## 题目简述

题目给出一串整数：

```text
[2864, 35, 1182, 37, 90, 28936, 8401, 821, 2957, 5677, 265, 7037, 40933, 29415, 92]
```

并提示它们是 LLM tokenizer 产生的 token。整数不是字符码点，而是某个 BPE 词表中的 token ID；只有选择与编码端相同的词表，才能无损还原原文。题目原 README 指向 OpenAI Tokenizer，因此优先测试 OpenAI 的常见编码。

## 解题过程

### 识别编码

OpenAI 的 [tiktoken](https://github.com/openai/tiktoken) 是可逆 BPE tokenizer，既支持
`encode(text)`，也支持 `decode(token_ids)`。题目发布于 2024 年，给出的 ID 在
`cl100k_base` 词表范围内；更直接的验证信号是：

- 前五个 token 在 `cl100k_base` 下组成 `UMDCTF{`。
- 最后一个 token 还原为 `}`。
- 中间 token 组合为可读的下划线分隔文本。

### 直接反向解码

```python
import tiktoken

token_ids = [
    2864, 35, 1182, 37, 90,
    28936, 8401, 821, 2957, 5677,
    265, 7037, 40933, 29415, 92,
]

encoding = tiktoken.get_encoding("cl100k_base")
print(encoding.decode(token_ids))
```

输出即为：

```text
UMDCTF{making_up_dune_lore_is_so_fun}
```

如果题面没有给出 tokenizer 家族，可依次尝试候选编码并用 flag 前后缀验证，但不能把 token ID 当 Unicode 或十进制 ASCII 直接转换：

```python
for name in ("r50k_base", "p50k_base", "cl100k_base"):
    try:
        text = tiktoken.get_encoding(name).decode(token_ids)
        if text.startswith("UMDCTF{") and text.endswith("}"):
            print(name, text)
    except Exception:
        pass
```

## 方法总结

- 核心技巧：把模型 token ID 当作特定 BPE 词表中的可逆表示，使用相同 tokenizer 的 `decode` 还原文本。
- 识别信号：一串中等大小整数、题面提到 LLM/tokenizer 且没有模型推理接口时，先检查 token ID 解码，而不是做密码爆破。
- 复用要点：token ID 没有跨词表的统一含义；必须确定编码名称，并用已知前缀、闭合括号和可打印性验证结果。
