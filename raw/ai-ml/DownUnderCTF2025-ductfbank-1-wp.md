# ductfbank 1

## 题目简述

题目提供的是 DownUnderCTF Bank 的聊天式柜员 Bobby。注册得到一个普通客户会话后，用户只能通过聊天请求其调用后端工具。源码显示，`create_account` 不只是创建记录：它会立刻调用 `giveBonus`，向新账户写入一笔金额为 1000 的开户奖励交易；奖励交易的 `description` 拼接了 `FLAG_BONUS`。因此本题的关键不是绕过鉴权或猜测 flag，而是让具备账户创建权限的模型正常完成一次开户。

## 解题过程

先在网站注册并登录，创建一个新对话。向 Bobby 提供任意账户昵称，例如“starter”，并明确请求创建个人账户。模型按其系统提示中的开户流程调用 `create_account`；工具实现的关键路径如下：

```ts
create_account: tool({
  parameters: z.object({ nickname: z.string() }),
  execute: async ({ nickname }) => {
    const account_number = await svc.createAccount(customerId, nickname);
    await svc.giveBonus(account_number);
    return { account_number };
  }
})

async giveBonus(account: string) {
  return this.db.transaction(async () => {
    const { id } = await this.db.query(
      'SELECT id FROM accounts WHERE number=?'
    ).get(account) as { id: number };
    await this.addTransaction(
      id,
      'DUCTF Bank',
      `Account opening bonus: ${FLAG_BONUS}`,
      1000
    );
  })();
}
```

工具返回的账户号会被 Bobby 告知用户。进入该账户的交易页，查看开户奖励一行的描述即可得到：

```text
DUCTF{1_thanks_for_banking_with_us_11afebf50e8cfd9f}
```

这里不需要诱导模型调用隐藏的 `flag` 工具；flag 已由正常业务逻辑写入自己账户可见的交易记录，且账户详情页面仅允许当前客户访问自己的账户。

## 方法总结

- 核心技巧：审计 Agent 工具的副作用，而不只盯着名称中含有 flag 的工具。这里“开户奖励”的交易备注就是泄露通道。
- 识别信号：聊天机器人被授予创建对象、发放优惠或生成凭证的能力时，应继续追踪工具调用的持久化数据及其在前端的展示位置。
- 复用要点：先走最小的合法业务流程；若 flag 随业务数据写入且用户本来就有读取该数据的权限，直接调用该流程比提示词绕过更稳定。
