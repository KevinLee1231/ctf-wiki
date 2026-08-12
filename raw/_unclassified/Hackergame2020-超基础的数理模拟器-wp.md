# Hackergame2020 超基础的数理模拟器 WP

## 题目简述

网站连续生成随机定积分，答对 400 题后返回 flag。服务端把当前积分的数值答案和累计分数放在加密 session 中，判定条件是提交值与标准值之差小于 $10^{-6}$；答错不会增加分数。

主障碍是批量解析 LaTeX、进行数值积分并维持 HTTP 会话，属于通用符号计算与自动化任务，暂归 `_unclassified`。

## 解题过程

题目生成器的被积函数由加减乘除、指数、三角函数、双曲函数、反正切、对数和平方根递归组成，积分上下限为正有理数。可以让 SymPy 解析页面中的 LaTeX，再用高精度数值积分计算结果。

必须使用同一个 `requests.Session()`：登录后 token、当前答案和进度都与 cookie 中的 session 绑定。下面脚本沿用官方页面结构，循环读取“还需答对的题数”和积分表达式：

```python
import re
from urllib.parse import quote_plus

import requests
from sympy import E, Integral, Symbol
from sympy.parsing.latex import parse_latex

BASE = "http://target.example:10190"
TOKEN = "替换为自己的 token"

x = Symbol("x")
e = Symbol("e")
session = requests.Session()
page = session.get(
    f"{BASE}/login?token={quote_plus(TOKEN)}",
    timeout=15,
).text

while "flag{" not in page:
    remaining_match = re.search(
        r'<h1 class="cover-heading">\s*(\d+)\s*题',
        page,
    )
    latex_match = re.search(r'<p>\s*\$(.*?)\$</p>', page, re.S)
    if not remaining_match or not latex_match:
        raise RuntimeError("页面结构或会话状态异常")

    remaining = int(remaining_match.group(1))
    latex = latex_match.group(1).strip()
    print("remaining:", remaining)

    # 页面形如 \int_{lower}^{upper} expression\,{d x}
    limits, integrand = latex.split(" ", 1)
    lower_text = limits[limits.index("_{") + 2:limits.index("}^{")]
    upper_text = limits[limits.index("^{") + 2:limits.rindex("}")]
    integrand = integrand.rsplit(r"\,{d x}", 1)[0]
    integrand = (
        integrand.replace(r"\left(", "(")
        .replace(r"\right)", ")")
        .replace(r"\,", "")
    )

    lower = parse_latex(lower_text).evalf(30)
    upper = parse_latex(upper_text).evalf(30)
    expression = parse_latex(integrand).subs(e, E)
    answer = Integral(expression, (x, lower, upper)).evalf(20)

    page = session.post(
        f"{BASE}/submit",
        data={"ans": str(answer)},
        timeout=30,
    ).text

print(page)
```

SymPy 的 LaTeX 解析器需要 `antlr4-python3-runtime`。如果某道题无法得到闭式表达式，`Integral(...).evalf()` 仍会做高精度数值求值；输出 20 位有效数字足以跨过服务端的 $10^{-6}$ 容差。

答满 400 题后得到：

```text
flag{S0lid_M4th_Phy_Foundation_<用户相关摘要>}
```

服务端源码表明后缀是当前 token 的 SHA-256 派生值，因此不同用户不会相同。

## 方法总结

自动化交互题应先读清状态保存位置和判定精度：本题若每次新建请求会话，分数与当前答案都会丢失。数学部分则要分开解析积分上下限和被积函数，统一替换 Sage 生成的 LaTeX 细节，并用高于判题阈值的精度计算。脚本遇到页面结构异常时立即停止，比盲目提交 400 次更容易定位问题。
