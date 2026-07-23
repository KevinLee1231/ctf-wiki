# Spice Harvest Data Recovery

## 题目简述

附件 `spice_analysis.csv` 表面是一份包含日期、地点、香料数量、纯度和采集队伍的表格，但文件头、分隔符和部分记录前缀都遭到破坏。目标不是根据 500 条业务数据计算统计量，而是从损坏文件中定位被伪装成异常记录的恢复线索，再解出其中的密文。

## 解题过程

先按原始字节检查文件，而不是直接交给电子表格软件。文件开头是：

```text
00 ff fe 23 c2 9f 70 54 ...
```

`00 ff fe` 不是与后续内容一致的有效 UTF-8 文件头；去掉这 3 个字节后，其余主体可以按 UTF-8 解码。表格中还混有逗号、分号以及重复出现的 10 字符乱码前缀，普通 CSV 解析器会把这些内容误判成畸形行。

在原始字节中搜索以 `# ` 开头、后接长十六进制串的异常行，可以找到唯一候选。它位于 3 月 14 日附近，十六进制正文长度为 482 个字符，即 241 字节。下面的脚本直接提取该行，不需要先修复全部表格：

```python
import re
from pathlib import Path

raw = Path("spice_analysis.csv").read_bytes()
match = re.search(rb"# ([0-9A-Fa-f]+)\r?\n", raw)
assert match is not None

message = bytes.fromhex(match.group(1).decode()).decode()
print(message)
```

得到：

```text
# LOOK TO ONE OF EARTH'S GREATEST RULERS IN THE ANCIENT EMPIRE OF ROME. LET ONE EARTH YEAR BEFORE THE BEGINNING OF HIS DOMINION HELP YOU DECODE THAT WHICH IS SHROUDED BY HIS NAMESAKE: &|sr%uLdb4Cbfd09`55b?0`?0f9b0dA`4b0CbGbc=0A_HbC0_GbC0c==N
```

“古罗马统治者”及“以他的名字命名”指向 Caesar 型轮换。密文不只含字母，还包含标点和数字，因此不是只作用于 26 个字母的传统 Caesar cipher，而是在 ASCII 可打印区间 `!` 到 `~` 的 94 个字符上循环移位。

提示中的年份表述存在历史口径歧义，不能单独作为可靠参数。直接枚举 94 种位移，并用 flag 格式做确定性筛选更稳妥：

```python
import re
from pathlib import Path

raw = Path("spice_analysis.csv").read_bytes()
hex_text = re.search(rb"# ([0-9A-Fa-f]+)\r?\n", raw).group(1)
decoded = bytes.fromhex(hex_text.decode()).decode()
ciphertext = decoded.split(": ", 1)[1]

for shift in range(94):
    candidate = "".join(
        chr(33 + (ord(char) - 33 + shift) % 94)
        if 33 <= ord(char) <= 126
        else char
        for char in ciphertext
    )
    if candidate.startswith("UMDCTF{") and candidate.endswith("}"):
        print(shift, candidate)
```

唯一命中是位移 47。由于 47 恰好是 94 的一半，ROT47 正向和反向使用同一变换：

```text
47 UMDCTF{53cr375_h1dd3n_1n_7h3_5p1c3_r3v34l_p0w3r_0v3r_4ll}
```

## 方法总结

这题的关键是把损坏 CSV 当作原始数字证据，而不是急于把每一行整理成规整表格。无效文件头、混合分隔符和重复乱码前缀都是干扰；真正有信息量的是唯一的长十六进制注释行。先做十六进制解码，再根据字符集识别并枚举 ROT47，即可建立完整且可复现的取证链。
