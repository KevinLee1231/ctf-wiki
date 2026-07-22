# safer-telegram-bot-2

## 题目简述

第二题把 `root` 用户的 Telegram user ID 固定为 `777000`。bot 收到 `/iamroot` 时会检查发送者 ID；只有消息看起来由 `777000` 发出，才会返回 `flag2`。

`777000` 同时是 Telegram 系统服务账号使用的历史 ID。2021 年的 [python-telegram-bot issue #2810](https://github.com/python-telegram-bot/python-telegram-bot/issues/2810)记录过一个危险现象：在群中“以频道身份发送”的消息会被旧版库识别为用户 `777000`，导致仅凭该 ID 判断管理员的 bot 产生越权。ACTF 2022 利用了同一类身份混淆：频道消息被自动转发到关联讨论组时，题目 bot 看到的发送者可满足 `777000` 检查。

[Telegram 的 Discussion Groups 文档](https://core.telegram.org/api/discussion)说明，频道可以绑定一个讨论 supergroup；每条频道帖子会自动转发到该群，并由这条转发消息建立评论线程。因此基本攻击模型是：让 bot 留在频道的讨论组，再从频道发布 `/iamroot`。

障碍在于 bot 监听 `my_chat_member`。一旦发现自己进入 ID 以 `-100` 开头的 supergroup 或 channel，它先发送警告，随后立即退群：

```js
bot.on("my_chat_member", async (update) => {
  if (String(update.chat.id).startsWith("-100")) {
    await sendMessage(update.chat.id, "This bot is not allowed to join groups");
    await bot.leaveChat(update.chat.id);
    return;
  }
});
```

所以题目真正要解决的是“如何阻止或绕过退群回调”。官方给出三条预期路线，比赛中还发现了一条利用异常控制流的非预期路线。

## 解题过程

### 1. 确定性路线：污染事件分发所依赖的原型

bot 提供 `/addkw key reply` 命令，把回复函数写入 `user1.keywordMap`：

```js
onText(/^\/addkw (\S+) (\S+)/, async (msg, match) => {
  const keyword = match[1];
  const reply = match[2];
  user1["keywordMap?." + keyword] = () => reply;
  await sendMessage(msg.chat.id, "success");
});
```

`user1` 的自定义 Proxy 把属性名按 `?.` 分段并逐层访问，核心路径解析逻辑如下：

```js
function resolvePath(target, prop) {
  const paths = prop.split("?.");
  let current = target;
  for (const path of paths) {
    current = current[path];
    if (!current) {
      return undefined;
    }
  }
  return current;
}
```

路径没有拒绝 `__proto__`、`prototype` 或 `constructor`。因此 `keywordMap?.__proto__?.chat_member` 会沿着 `keywordMap.__proto__` 到达 `Object.prototype`，并在其上写入 `chat_member`。实际命令是：

```text
/addkw __proto__?.chat_member 1
```

如果 Telegram 客户端把双下划线解释成格式标记，应先把 `__proto__` 选中并设置为等宽代码格式，确保 bot 收到的文本仍包含两个下划线。

为什么污染 `chat_member` 可以阻止退群？题目使用的固定版本 `node-telegram-bot-api` 会先从普通 `update` 对象逐个取字段，再用一串 `else if` 选择事件类型。其[对应提交中的 `processUpdate()` 源码](https://github.com/yagop/node-telegram-bot-api/blob/d28875154cf89d73e71407e53c7f7738073ca5ff/src/telegram.js#L613)抽出与本题有关的分支后，等价顺序如下：

```js
function processRelevantUpdate(update) {
  const pollAnswer = update.poll_answer;
  const chatMember = update.chat_member;
  const myChatMember = update.my_chat_member;

  if (pollAnswer) {
    this.emit("poll_answer", pollAnswer);
  } else if (chatMember) {
    this.emit("chat_member", chatMember);
  } else if (myChatMember) {
    this.emit("my_chat_member", myChatMember);
  }
}
```

从 Telegram JSON 解析出的 `update` 是普通对象。当它没有自有 `chat_member` 字段时，属性查找会退到已经被污染的 `Object.prototype.chat_member`，得到一个真值；分发器于是先进入 `chat_member` 分支，不再检查真实存在的 `my_chat_member`。题目只在 `my_chat_member` 上注册了退群逻辑，所以 bot 会留在群中。

完整操作顺序如下：

1. 私聊 bot，发送上述 `/addkw` 命令并确认返回 `success`；
2. 创建频道和一个 supergroup，把 supergroup 设为频道的 Discussion Group；
3. 将 bot 加入这个讨论组；由于事件被误分发，bot 不会执行 `leaveChat`；
4. 在频道发布 `/iamroot`；
5. 等待帖子自动转发到讨论组，bot 将其识别为来自 `777000` 并在群中返回 flag2。

这条路线只依赖题目 Node.js 进程和固定库版本，避免了精确卡 Telegram 转发时序，因此最适合作为主解。

### 2. 平台路线：先加入普通 group，再迁移为 supergroup

Telegram 历史上同时存在普通 `group` 与 `supergroup`。普通群的 chat ID 为负数，但不以 `-100` 开头；supergroup 和 channel 的 ID 才采用 `-100...` 形式。新建群最初是普通 group，在启用公开用户名、细粒度权限或关联频道等高级能力时会迁移为 supergroup，并获得新的 chat ID。

可利用迁移前后的差异：

1. 新建一个尚未迁移的普通 group；
2. 先把 bot 加入，此时 chat ID 不以 `-100` 开头，回调不会退群；
3. 再把该群关联到频道，使其迁移为 supergroup/Discussion Group；
4. 客户端询问是否让新成员看到历史消息时选择“不公开历史”；
5. 在频道发布 `/iamroot`，由自动转发触发 root 检查。

如果迁移时选择向 bot 公开旧历史，bot 可能重新收到成员状态 Update，并在新的 `-100...` chat ID 上触发退群。官方记录给出的替代办法是先发送至少 100 条普通消息，把最初的进群服务消息挤出可见历史，再进行关联。

### 3. 时序路线：利用两个 `await` 之间的窗口

退群回调并不是原子操作：

```js
await sendMessage(update.chat.id, "This bot is not allowed to join groups");
await bot.leaveChat(update.chat.id);
```

可先在已经关联讨论组的频道中发布 `/iamroot`，随后立刻邀请 bot 进入讨论组。频道到讨论组的自动转发存在延迟；理想顺序是：

1. bot 收到进群 Update，开始发送警告；
2. 警告发送完成、`leaveChat` 尚未完成；
3. 预先发布的 `/iamroot` 恰好转发进群；
4. bot 在仍有发言权限时把 flag2 发回群中。

如果先加 bot 再发频道消息，进群 Update 几乎必然先触发退群；如果 flag 回复发生在 `leaveChat` 完成之后，发送也会失败。该方法可行但窗口短，稳定性明显低于原型链污染。

### 4. 非预期路线：先禁言，让异常跳过 `leaveChat`

回调没有使用 `try...finally`。如果第一条 `await sendMessage(...)` 抛出异常，JavaScript 会直接退出异步函数，下一行 `leaveChat` 根本不会执行。

操作方法是：

1. 在讨论组中开启全员禁言，使新加入的普通成员无法发言；
2. 邀请 bot 入群；
3. bot 尝试发送退群警告，但因权限不足而失败并抛出异常；
4. 由于异常未捕获，bot 留在群里；
5. 解除禁言，再从关联频道发送 `/iamroot`。

`my_chat_member` 只在成员状态变化时触发；单纯解除群发言限制不会自动重跑刚才失败的回调，因此 bot 能继续留在讨论组并发送 flag。这条路线不需要污染或竞速，但依赖 bot 加群时的具体权限设置。

实际 flag 存放在比赛服务的 `flag2` 文件或环境中，仓库只保留机制说明，没有 flag 明文，不能根据题名或占位内容猜写。

## 方法总结

本题首先利用了身份语义混淆：`777000` 既被业务逻辑当作可信 root，又会在 2022 年的特定频道/讨论组消息路径中代表 Telegram 系统转发。仅检查 `from.id === 777000` 无法区分“真正的系统账户私聊”与“频道消息的兼容表示”。这类依赖第三方平台特殊 ID 的授权规则本身就不稳健。

四种路线分别破坏了退群机制的不同前提：

- group 迁移利用 chat ID 与成员事件在迁移前后的不一致；
- 原型链污染改变库对 Update 类型的判断，是最确定的代码级路线；
- 条件竞争利用异步转发与两个 `await` 的时间窗口；
- 全员禁言利用未捕获异常中断控制流。

修复时应同时处理三个层面：路径解析必须拒绝 `__proto__`、`prototype` 与 `constructor`；事件分发和权限判断只读取自有属性，例如使用 `Object.hasOwn(update, "chat_member")`；退群动作应放入可控的异常处理或 `finally` 中。更根本的是，不应把 Telegram 的特殊账号 ID 当作唯一的 root 身份证明。
