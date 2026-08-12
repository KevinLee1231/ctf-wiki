# Hackergame2020 从零开始的记账工具人 WP

## 题目简述

题目提供一份账单，每行包含用中文财务大写表示的单价和购买数量。需要把各行金额换算为人民币分，计算总价，并以 `flag{整数部分.两位小数}` 的格式提交。

附件由用户 token 作为随机种子生成，因此不同用户的账单和最终数值可能不同。核心是表格读取、中文金额解析和精确求和，属于通用编程任务，暂归 `_unclassified`。

## 解题过程

财务数字的基本映射是 `零壹贰叁肆伍陆柒捌玖` 到 `0` 至 `9`，整数部分还会出现 `拾、佰、仟` 等单位，小数部分使用 `角、分`。不要用二进制浮点数累加金额；直接把每个单价转换成整数“分”，最后再格式化，能完全避免 `0.1 + 0.2` 一类误差。

下面使用 `openpyxl` 读取原始 `.xlsx`，用 `cn2an` 只解析各段中文整数，再自行组合为分：

```python
from openpyxl import load_workbook
import cn2an


def chinese_integer(text):
    if not text or text == "零":
        return 0
    return int(cn2an.cn2an(text, "smart"))


def rmb_to_cents(text):
    text = str(text).strip().replace("整", "")
    yuan_text, sep, tail = text.partition("元")
    if sep:
        cents = chinese_integer(yuan_text) * 100
    else:
        # 金额不足一元时，整段都是“角/分”部分。
        cents = 0
        tail = yuan_text

    if "角" in tail:
        jiao_text, tail = tail.split("角", 1)
        cents += chinese_integer(jiao_text) * 10
    if "分" in tail:
        fen_text = tail.split("分", 1)[0]
        cents += chinese_integer(fen_text)
    return cents


book = load_workbook("bills.xlsx", data_only=True, read_only=True)
sheet = book.active

total_cents = 0
for price_text, count in sheet.iter_rows(min_row=2, values_only=True):
    total_cents += rmb_to_cents(price_text) * int(count)

print(f"flag{{{total_cents // 100}.{total_cents % 100:02d}}}")
```

可额外抽查几行：例如 `壹佰贰拾叁元肆角伍分` 应得到 12345 分，乘以数量后仍保持整数。最终运行结果就是当前账单对应的 flag；不应照抄其他选手题解中的金额，因为服务端按 token 生成数据。

## 方法总结

这类账单题的风险不在求和，而在输入表示和金额精度。正确做法是先确定列含义，再把中文金额按 `元、角、分` 分段解析，内部统一使用整数分。只有最后输出时再插入小数点，既满足“两位小数”的格式要求，也避免浮点误差和不同舍入规则造成的错答。
