# llm compression

## 题目简述

题目给出一个 30 字节的二进制 `flag.txt` 和约 950 KB 的 `params.npz`。后者包含
Transformer 的嵌入层、四组多头注意力参数、前馈层和 LayerNorm 参数；前者不是损坏文本，而是用该语言模型的逐字节概率分布配合算术编码压缩后的 flag。

语言模型并不直接“记住答案”。解压端必须加载完全相同的权重，并在每一步根据已经恢复的前缀预测下一个字节的 256 项概率分布，再让算术解码器选择唯一符号。

## 解题过程

### 还原上游压缩器

README 指向 DeepMind 的
[Language Modeling is Compression](https://github.com/google-deepmind/language_modeling_is_compression) 仓库。该项目的关键机制是：

- Transformer 输出下一个字节的对数概率，字母表大小为 256。
- 概率经规范化后作为算术编码的频率表。
- 算术编码基数为 2、内部精度为 32 位。
- 解码必须逐字节重新运行模型，因为第 $i$ 个分布依赖已恢复前缀
  $x_0,\ldots,x_{i-1}$。

因此不需要重新训练模型。将题目给出的 `params.npz` 放在上游项目工作目录，使
`compressors/language_model.py` 的 `_retrieve_model_params()` 能直接加载它。

### 调用模型算术解码

上游 `decompress()` 的核心流程是先把压缩字节转为比特流，再初始化算术解码器。模型右移输入以预测下一符号，所以实现会先放入一个值无关的 dummy token：

```python
sequence = np.empty((1,), dtype=np.uint8)
probs = np.exp(predict_fn(sequence[None])[0])

for index in range(uncompressed_length):
    symbol = decoder.decode(
        normalize_pdf_for_arithmetic_coding(probs[index])
    )
    sequence = np.insert(sequence, -1, symbol)
    probs = np.exp(predict_fn(sequence[None])[0])

plaintext = sequence[:-1].tobytes()
```

算术编码输出会补零到整字节，解压函数需要知道补了多少位；原文件长度也不是压缩流自描述字段。本题 flag 长度为 35 字节，可枚举仅 8 种补位数量，并用标准 flag 外形验证：

```python
from compressors.language_model import decompress

compressed = open("flag.txt", "rb").read()

for padded_bits in range(8):
    candidate = decompress(
        compressed,
        num_padded_bits=padded_bits,
        uncompressed_length=35,
    )
    if candidate.startswith(b"UMDCTF{") and candidate.endswith(b"}"):
        print(candidate.decode())
        break
```

只有与压缩端一致的补位数会让算术区间和后续模型上下文持续对齐，结果为：

```text
UMDCTF{holy_compression_with_ml!!!}
```

## 方法总结

- 核心技巧：把语言模型视为自适应概率源，使用同一模型权重驱动算术解码器逐字节重建原文。
- 识别信号：附件同时包含 Transformer 参数和短小高熵二进制流，题面又明确提到“language modeling is compression”时，应寻找模型概率与熵编码的组合，而不是把 `.npz` 当普通隐藏压缩包。
- 复用要点：模型结构、权重、概率量化、算术编码精度、原文长度和末尾补位必须全部一致；任何一项错位都会让后续字符迅速雪崩。
