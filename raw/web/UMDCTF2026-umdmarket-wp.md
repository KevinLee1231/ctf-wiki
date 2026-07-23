# umdmarket

## 题目简述

题目通过 HTTP/3 WebTransport 提供二进制预测市场协议。报价 token 包含 `seq`、ticker、YES price 和截断 HMAC，交易接口会验证签名及序列号新鲜度。达到 10000000 个内部余额单位后可以买 flag。

漏洞位于丢包重传接口 `RESEND`：它取出历史价格，却把 token 的序列号改为当前 `seq` 并重新签名。于是历史价格被包装成了“新鲜报价”。

## 解题过程

先注册、登录，订阅全部 ticker，并收集 QUOTE datagram。服务端保存 500 tick 的 resend tape，每 tick 100 ms，所以历史窗口约为 50 秒；参考脚本预热约 55 秒，并只使用最近 45 秒的本地记录，避免槽位已被覆盖。

对每个 ticker 比较历史价与当前价：

- 若当前 YES 价高于某个历史低价，重传低价 token，`BUY_YES`，等待冷却后用最新 token `SELL_YES`；
- 若当前 YES 价低于某个历史高价，则历史时刻的 NO 价更低，重传高 YES 价 token，`BUY_NO`，随后用最新 token `SELL_NO`。

以 YES 为例，`RESEND(old_seq,ticker)` 返回：

```text
seq   = current_seq
price = old_price
hmac  = HMAC(current_seq || ticker || old_price)
```

交易处理器只检查 `session_seq - token_seq <= MaxAge`，并直接采用 token 内价格成交，所以第一条腿能以历史低价买入。仓位数量取当前余额可负担数量的约 90%，留出取整空间；每次成交后等待至少 5 秒冷却，再完成反向交易。

循环选择价差最大的机会，直到账户余额不低于 10000000，然后发送 `BUY_FLAG`：

```text
UMDCTF{fu7ur3_j4n3_s7r33t_h1r3}
```

## 方法总结

签名只能证明字段由服务端生成，不能证明这些字段在业务语义上彼此一致。重传逻辑把“旧价格”和“当前序列号”重新签成一个合法但不应存在的组合，破坏了报价的新鲜度绑定。正确修复是保留原始序列号，或在重传时返回当前价格，不能只重新计算 HMAC。
