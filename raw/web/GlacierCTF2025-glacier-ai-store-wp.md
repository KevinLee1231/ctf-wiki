# GlacierCTF 2025 Glacier AI Store

## 题目简述

题目是一个 PHP 商店。新用户余额为 1，商品价格依次为 stone 1、water 10、jar 100、flag 1000；购买 flag 后，订单描述会直接显示 `/flag.txt`。

取消订单的代码先把商品价格退回余额，再缓慢输出用户提供的取消原因，最后才从数据库删除商品。PHP 默认在客户端断开且脚本再次输出时终止请求，因此可在退款完成后主动断开，使删除步骤永远不执行，反复出售同一件商品。

## 解题过程

### 1. 定位非原子的出售流程

关键代码顺序为：

```php
if (!hasUserProduct($loginID, $sell)) goto product_list;
increaseBalance($loginID, $products[$sell]["price"]);
if (isset($reason) && strlen($reason) > 0)
    respondText($USER, $reason);
sellProduct($loginID, $sell, $reason);
```

`respondText` 每输出一个字符都等待 50 ms，并调用 `ob_flush()`、`flush()`。它位于“余额已增加”和“商品已删除”之间，给了客户端一个稳定的中止窗口。PHP 配置没有启用 `ignore_user_abort(true)`；客户端关闭连接后，后续 flush 检测到断开，脚本终止。

这不是并发 race：一次请求内的执行顺序已经足够。目标是让前半段数据库更新提交，后半段不发生。

### 2. 在标记字节出现时立即断开

购买最便宜的 stone 后，发送流式 POST：

```python
with session.post(
    products_url,
    data={"sell": "stone", "reason": "\x0c" + "A" * 100},
    stream=True,
) as response:
    for byte in response.iter_content(chunk_size=1):
        if byte == b"\x0c":
            response.close()
            break
```

`0x0c`（form feed）是攻击者放在 reason 开头的同步标记。收到它说明服务已执行 `increaseBalance` 并进入慢速输出；此时关闭响应，随后输出长串 `A` 时服务发现连接已断。请求结束后再次访问账户页，应看到余额增加，但 stone 仍在订单列表中。这个状态差分就是漏洞成功的验证，而不是只根据客户端异常猜测。

### 3. 逐级放大余额

利用价格的十进制阶梯：

1. 花 1 买 stone，异常出售 10 次，余额到 10，商品仍保留；
2. 花 10 买 water，异常出售 10 次，余额到 100；
3. 花 100 买 jar，异常出售 10 次，余额到 1000；
4. 正常购买 flag，在订单 overview 中读取 `ordered_desc`。

源码实例得到：

```text
gctf{pHP_G3tZ_W3iRD_Wh3n_y0U_D!SsC0nN3Ct_m0K9Pa5shMpOO3Mq}
```

## 方法总结

本题是“响应生命周期参与业务事务”的逻辑漏洞。危险点不只是慢速输出，而是退款和删除没有放在同一个原子事务中，还在二者之间执行可能因客户端断开而终止的 I/O。修复应在事务内同时验证所有权、删除商品并增加余额，成功提交后再生成响应；或者至少在关键业务完成前禁止 user-abort 影响执行。客户端 exploit 也应在每次断开后读取账户页验证余额和持有状态，避免因代理缓冲导致误判。
