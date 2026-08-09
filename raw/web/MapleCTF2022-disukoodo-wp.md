# disukoodo

## 题目简述

题目运行一个 Discord 机器人，普通用户只能使用受限命令，premium 用户则可以指定重复次数。关键缺陷不是 Discord 协议本身，而是机器人会处理自己通过 `echo` 发出的新消息，并且 premium 注册命令没有可靠校验消息的真实发起者，形成消息级访问控制绕过。

## 解题过程

先私信机器人并在消息中提及它。普通账号可使用 `echo`，于是让机器人复述一条再次提及机器人的管理命令：

```text
@beepboop echo @beepboop registerprem @attacker
```

第一层命令由攻击者发送，但第二层消息的作者是机器人自己。事件处理器再次解析该消息时，把它当作可信来源执行 `registerprem`，从而将攻击者登记为 premium。

premium `echo` 接受次数和内容。让它尝试输出超过 Discord 单条消息 2000 字符的文本，例如：

```text
@beepboop echo 100 12345678901234567890
```

Discord API 返回 `HTTPException`。机器人的通用异常处理器错误地把调试上下文回显给用户，其中包含 `bot.guilds` 对象及服务器标识；题目把目标 flag 设置为隐藏 guild 的名称，因此从泄露列表即可读到：

```text
maple{ch4r_l1m1ts_4cc3ss_c0ntr0l_4nd_l0vely_m4rkd0wn}
```

## 方法总结

机器人转发或复述的消息不能继承原命令的信任级别，更不能仅凭“作者是机器人自身”开放管理操作。还应在调用 Discord API 前检查长度和重复次数，并确保异常处理只返回固定错误文本。尽管交互载体是 Discord，本题的决定性机制是应用命令解析、身份传递和错误信息泄露，归入 Web 更合适。
