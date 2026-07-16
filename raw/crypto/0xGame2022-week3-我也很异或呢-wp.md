# week3我也很异或呢

## 题目简述

附件 `flag` 被短密钥循环异或。密文的周期统计显示候选密钥长度为 6，结合十六进制视图中的重复模式和题目格式，可以恢复密钥 `0xGame`。解密结果是一份 ZIP，其中的 `flagishere.Q` 是按键精灵脚本。

## 解题过程

循环异或满足：

$$
C_i=P_i\oplus K_{i\bmod |K|}.
$$

由于同一密钥字节每 6 字节重复使用，各列的字符分布会出现明显周期。`xortool` 的关键输出可直接转写为：

```text
The most probable key lengths:
3: 14.2%
6: 19.0%
9: 11.4%
12: 12.7%
...
Key-length can be 3*n

1 possible key(s) of length 6:
0xGame
```

长度 6 的评分最高，且候选 `0xGame` 可读，因此先用它验证文件头。

不依赖工具也可以从十六进制视图观察到周期性，并用预期文件头验证候选。使用 `0xGame` 对全部字节再次异或：

```python
from pathlib import Path

cipher = Path("flag").read_bytes()
key = b"0xGame"

plain = bytes(
    value ^ key[index % len(key)]
    for index, value in enumerate(cipher)
)

assert plain.startswith(b"PK\x03\x04")
Path("decoded.zip").write_bytes(plain)
```

`PK\x03\x04` 断言通过，说明密钥和相位均正确。解压得到 `flagishere.Q`；`.Q` 是按键精灵脚本格式，将其导入按键精灵并运行后，脚本创建文本并输出：

```text
0xGame{4e09d99d-f1f5-4287-a6ca-7467d2ed6ec0}
```

## 方法总结

短密钥循环异或的弱点是周期重复：先用统计或重复模式确定密钥长度，再借助已知文件魔数验证密钥内容和起始相位。解密后必须继续识别新文件格式，ZIP 只是中间层，真正结果位于按键精灵脚本中。原解只展示截图而漏写最终 flag，本 WP 已补齐完整结果；外部工具链接不是理解攻击所必需，因此不保留。
