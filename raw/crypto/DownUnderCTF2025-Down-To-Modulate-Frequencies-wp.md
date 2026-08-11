# Down To Modulate Frequencies!

## 题目简述

附件是一串没有分隔符的十进制数字。题名首字母组成 `DTMF`，提示先按双音多频键盘解码；但附件记录的不是音频，而是每个按键低频与高频之和形成的四位数。还原按键序列后，还要按老式手机的 multi-tap 规则把重复数字转换为字母。

## 解题过程

### 还原 DTMF 按键

标准键盘中，每个按键由一组低频和高频确定。题目把二者相加，例如：

```text
4 -> 770 + 1209 = 1979
5 -> 770 + 1336 = 2106
6 -> 770 + 1477 = 2247
# -> 941 + 1477 = 2418
```

所有和值都是四位数，所以把附件每四位切分，再用完整映射表反查，就能得到只含数字与 `#` 的序列。`#` 是相邻 multi-tap 字符组的分隔符。

### 解码老式手机多按键输入

对每一组连续相同数字计数，并在对应按键字符表中取第 $n$ 个字符。例如 `666` 为 `o`、`66` 为 `n`、`555` 为 `l`、`999` 为 `y`。可复现的核心代码如下：

```python
from pathlib import Path

dtmf_sum = {
    "1906": "1", "2033": "2", "2174": "3", "2330": "A",
    "1979": "4", "2106": "5", "2247": "6", "2403": "B",
    "2061": "7", "2188": "8", "2329": "9", "2485": "C",
    "2150": "*", "2277": "0", "2418": "#", "2574": "D",
}
keypad = {
    "2": "abc", "3": "def", "4": "ghi", "5": "jkl",
    "6": "mno", "7": "pqrs", "8": "tuv", "9": "wxyz",
}

data = Path("dtmf.txt").read_text().strip()
chunks = [data[i:i + 4] for i in range(0, len(data), 4)]
digits = "".join(dtmf_sum[chunk] for chunk in chunks)

message = ""
for group in digits.split("#"):
    if group:
        assert len(set(group)) == 1
        message += keypad[group[0]][len(group) - 1]
```

得到：

```text
onlyninetieskidswillrememberthis
```

按题目要求包裹后，flag 为：

```text
DUCTF{onlyninetieskidswillrememberthis}
```

## 方法总结

- 核心技巧：先用 DTMF 频率和反查数字键，再用旧式手机 multi-tap 码表解码字母。
- 识别信号：标题直接形成 `DTMF`，输入由定长四位数字组成，提示又反复提及电话按键和智能手机之前的短信输入。
- 复用要点：不要把“组合频率”误当成真实音频采样；先确认题目使用的是频率对、频率和值还是频谱峰值。多按键编码还需要明确字符组边界，本题用 `#` 分隔。
