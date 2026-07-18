# week2ENCODE

## 题目简述

题目把多种网络文本编码连续嵌套：与佛论禅、中文字符替换后的 Ook、Brainfuck，以及 Microsoft Script Encoder 编码。解题重点是依据每一层的字符特征识别下一层。

## 解题过程

第一层的佛经样式文本按“新与佛论禅”规则还原后，得到反复出现的句子：

```text
既见未来，为何不拜？
```

这并不是最终明文。题目提示要求把这些中文字符按给定替换关系恢复成 Ook 标记。Ook 使用 `Ook.`、`Ook?`、`Ook!` 两两组合来表示八条 Brainfuck 指令，标准对应关系如下：

| Ook 组合 | Brainfuck |
| --- | --- |
| `Ook. Ook?` | `>` |
| `Ook? Ook.` | `<` |
| `Ook. Ook.` | `+` |
| `Ook! Ook!` | `-` |
| `Ook! Ook.` | `.` |
| `Ook. Ook!` | `,` |
| `Ook! Ook?` | `[` |
| `Ook? Ook!` | `]` |

将恢复出的 Ook 转为 Brainfuck 并执行后，会得到以

```text
#@~^
```

开头的文本。`#@~^` 是 Microsoft Script Encoder 的编码头，常见完整载荷以 `^#~@` 收尾；它不是随机乱码，也不是普通 Base 编码。最后对该 VBE/JSE 编码块进行 Script Encoder 解码即可得到 flag。

当前仓库没有保留本题附件、中文到 Ook 的具体替换表或最终 flag，官方 Week 2 PDF 也只记录了上述层次。因此现有证据能确认完整识别链，但不能可靠补写具体中间密文与最终答案。

## 方法总结

本题的可靠识别链是“与佛论禅 $\to$ 中文替换 $\to$ Ook $\to$ Brainfuck $\to$ Script Encoder”。处理套娃编码时应记录魔数、字符集和每层输出；源材料缺失时应保留已确认机制，而不是凭相似题目猜测 flag。
