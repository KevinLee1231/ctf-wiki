# Joe Dohn

## 题目简述

题目提供一个 Discord Bot。普通成员能够使用 `!send_msg <message>` 把任意文本转发到不可见的内部请求频道；只有持有管理员角色的成员才能执行 `!send_secret <username>`，让 Bot 私信 secret。

虽然交互载体是 Discord，本质漏洞仍是服务端命令处理中的信任边界错误：Bot 把用户文本原样发布为自己的消息，又会把自己的命令消息重新当作受信任输入执行，因此归入 Web 与应用逻辑方向。

## 解题过程

源码的关键分支为：

```python
if message.content.startswith("!"):
    cmd = message.content.split(" ")
    match cmd[0]:
        case "!send_secret":
            role = message.guild.get_role(role_id)
            if role in message.author.roles:
                username = " ".join(cmd[1:])
                member = guild.get_member_named(username)
                if member:
                    await member.send(flag)

        case "!send_msg":
            content = " ".join(cmd[1:])
            request_channel = bot.get_channel(request_id)
            await request_channel.send(content)
```

程序只在“非命令消息”分支中检查：

```python
if message.author != bot.user:
    ...
```

命令分支没有排除 Bot 自己发送的消息。与此同时，Bot 自身拥有管理员角色。因此，在个人频道发送：

```text
!send_msg !send_secret <自己的完整 Discord 用户名>
```

会形成以下调用链：

1. 普通用户有权调用 `!send_msg`。
2. Bot 把参数 `!send_secret <用户名>` 原样发送到内部请求频道。
3. 该新消息的作者变成 Bot 自身。
4. `on_message` 再次收到这条以 `!` 开头的消息，并进入命令分支。
5. `!send_secret` 检查的是消息作者角色；Bot 自身具有管理员角色，校验通过。
6. Bot 向指定用户私信 flag。

最终得到：

```text
N0PS{d15c0Rd_c0mM4nD_1nJ3c710n}
```

## 方法总结

- 核心技巧：利用消息转发造成命令注入，让低权限输入在高权限 Bot 身份下被二次解析。
- 识别信号：存在“转发任意消息”功能、转发后作者身份发生变化、事件处理器会消费自己发出的消息，且权限检查依赖消息作者。
- 复用要点：修复时需要同时阻断自消息递归、限制转发内容并避免把展示文本重新送入命令解释器；只在某一个普通消息分支检查 `author != bot.user` 并不能保护命令分支。
