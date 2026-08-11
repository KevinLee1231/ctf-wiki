# ductfbank 3

## 题目简述

题目仍使用 Bobby 的聊天柜员，但这次 flag 存在 SQLite 的 `flags(flag)` 表中。决定性链条有两层：模型的 `get_account_details` 工具把其 `number` 参数交给 `BankService.getAccount`；后者将该值直接插入 SQL 字符串，导致注入。模型的系统提示会拒绝来自用户消息的 SQL 字样，因此还要利用可修改的账户昵称，让查询指令经 `list_accounts` 作为“可信工具输出”回流给模型。虽然利用了 SQL 注入，能否让 Agent 以攻击载荷调用工具才是取得结果的必要障碍，故归入 AI/ML。

## 解题过程

源码中的危险查询和结果包装如下：

```ts
const stmt = this.db.query(`
  SELECT id, customer_id, number, nickname, balance, created_at
  FROM accounts
  WHERE number='${number}'
`);
const result = await stmt.get();
const { nickname, balance, customer_id } = result;
return { nickname, number, balance, customer_id };
```

先将自己的账户昵称改为一段要求 Bobby 对下列“账户号”调用 `get_account_details` 的文字，再在聊天中要求其查看自己的账户。这样模型先通过 `list_accounts` 读到昵称，再将其中的载荷传给详情工具；这是为了避开系统提示对用户直接提交 SQL 的拦截，而不是绕过应用层的账户权限检查。

要先确定查询可返回的列数。原查询有六列，可用 `sqlite_master` 将表名放到会回显的第四列 `nickname` 中，例如：

```sql
' OR 1=0 UNION SELECT 1,1,1,name,1,1 FROM sqlite_master --
```

返回表中出现 `flags` 后，读取 flag：

```sql
' OR 1=0 UNION SELECT 1,1,1,flag,1,1 FROM flags --
```

前面的 `OR 1=0` 使原表分支不产生记录，`UNION SELECT` 则构造一行六列结果。当前仓库的工具包装只把 `nickname`、`number`、`balance` 和 `customer_id` 交给模型，因此 flag 必须放在第四列 `nickname` 才会回显。官方说明中将 flag 放入第六列的示例，在本仓库代码里会落在未返回的 `created_at` 字段，不能作为可见结果使用。

模型成功执行工具后，`nickname` 字段回显：

```text
DUCTF{3_you_hacked_the_mainframe_05598352e7d6a61b21e4}
```

## 方法总结

- 核心技巧：把模型的间接提示注入与 SQL `UNION` 查询组合；前者负责让工具接受恶意参数，后者负责把目标表的值伪装成账户记录。
- 识别信号：当 Agent 工具把字符串参数交给使用模板拼接的数据库查询时，应同时检查模型提示词能否阻断直传，以及用户控制的持久化字段是否会经工具结果回流。
- 复用要点：`UNION` 的列数、类型和回显位置必须与最终工具包装一致，不能只看底层 SQL。修复时应参数化 `WHERE number=?`，并在服务端对工具参数做所有权与格式校验，不能把 SQL 过滤交给模型。
