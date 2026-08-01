# BYUCTF 2023 - Hexadecalingo

## 题目简述

题面由 17 个不同语言的单词组成。逐词翻译只会得到一句干扰提示；真正的隐藏层是每个单词所属语言名称的首字母。

## 解题过程

原句及语言映射如下：

| 单词 | 语言 | 首字母 |
| --- | --- | --- |
| `আপনি` | Bengali | B |
| `געדאַנק` | Yiddish | Y |
| `оцей` | Ukrainian | U |
| `byl` | Czech | C |
| `தி` | Tamil | T |
| `drapeau` | French | F |
| `но` | Macedonian | M |
| `este` | Romanian | R |
| `mwy` | Welsh | W |
| `ମେଟା` | Odia | O |
| `decât` | Romanian | R |
| `quod` | Latin | L |
| `ik` | Dutch | D |
| `dymuno` | Welsh | W |
| `tú` | Irish | I |
| `ރަނގަޅު` | Dhivehi | D |
| `õnne` | Estonian | E |

首字母连接为：

```text
BYUCTFMRWORLDWIDE
```

前六个字符是比赛前缀，余下部分即答案：

```text
byuctf{mrworldwide}
```

## 方法总结

多语言 OSINT 不应止于翻译结果，还要观察语言本身是否构成元数据。题面翻译后的句子直接提示“more meta”，说明应从语言名称、文字系统或国家信息中再取一层。
