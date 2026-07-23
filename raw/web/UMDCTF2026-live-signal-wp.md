# live-signal

## 题目简述

Next.js 服务端 action `signalsByAnalyst` 声明参数类型为只含 `handle` 和 `tier` 的 `AnalystFilter`，但 TypeScript 类型在运行时不存在。服务端把客户端对象直接放入 Prisma 的 relation filter：

```typescript
where: {
  analyst: author,
  publishedAt: { lte: new Date() },
}
```

攻击者可注入完整的 `AnalystWhereInput`，并用 `signals.some` 对尚未发布的 draft 建立布尔 oracle。

## 解题过程

提交如下结构：

```json
{
  "author": {
    "handle": "quant_sable",
    "signals": {
      "some": {
        "publishedAt": {"gt": "当前时间"},
        "side": "YES",
        "ticker": {"startsWith": "KX"},
        "contractPrice": {"gte": 40}
      }
    }
  },
  "limit": 1
}
```

外层 `publishedAt <= now` 只限制最终返回的历史 signal，并不会限制嵌套 `signals.some` 检查的关联行。只要未来 draft 满足猜测，`quant_sable` 的某条历史 signal 就会被返回；数组非空即为真。

按以下顺序提取字段：

1. 测试 `side == YES`，否则为 `NO`；
2. 在 `A-Z0-9-` 中逐字符测试 `ticker.startsWith(prefix)`；
3. 用 `contractPrice.gte(mid)` 在 1 到 99 间二分；
4. 在十六进制字符集中逐位测试 `signalHmac.startsWith(prefix)`，恢复 32 字符 HMAC。

当前源码的限速是每 IP 每分钟 600 次，参考脚本每次请求间隔约 110 ms，并在 429 后退避。最后向 `verifyInsider` 提交 `ticker`、`side`、`contractPrice` 和 `signalHmac` 四个精确字段，得到：

```text
UMDCTF{z0d_1s_f0r_sc4r3dy_c4ts}
```

## 方法总结

静态类型不能充当运行时输入校验。把客户端对象原样传给 ORM 会把 ORM 的完整查询语法暴露给攻击者；关联存在性过滤又把不可见行变成真假 oracle。应显式构造允许字段的新对象，或在 action 边界使用运行时 schema 严格拒绝额外键和嵌套对象。
