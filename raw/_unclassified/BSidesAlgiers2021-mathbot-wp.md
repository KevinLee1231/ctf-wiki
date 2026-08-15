# MathBot

## 题目简述

MathBot 通过 Discord 连续发送 50 道两个 64 位正整数相乘的问题，每题必须在 3 秒内回复纯数字答案。全部答对后才发送 flag。

这是一道协议自动化与大整数计算题，没有稳定对应的安全漏洞方向，因此放在 _unclassified，而不是因为源码使用 Python 就归入 crypto 或 pwn。

## 解题过程

挑战端每轮生成：

~~~python
a = random.randint(1, 2**64)
b = random.randint(1, 2**64)
equation = f"{a} * {b}"
answer = a * b
~~~

人工无法稳定在 3 秒内完成 50 轮。官方方案运行另一个 Discord bot，过滤挑战 bot 的用户 ID，读取消息后立即计算并在同一频道回复。官方 solver 直接 eval 消息文本；因为输入来自另一 bot 且语法固定，比赛中可以工作，但没有必要保留这种额外代码执行面。更安全的解析逻辑如下：

~~~python
def solve_equation(text):
    left, operator, right = text.split()
    if operator != "*":
        raise ValueError("unexpected operator")
    if not left.isdecimal() or not right.isdecimal():
        raise ValueError("unexpected operands")
    return int(left) * int(right)

@bot.event
async def on_message(message):
    if message.author.id != CHALLENGE_BOT_ID:
        return
    try:
        result = solve_equation(message.content)
    except ValueError:
        print(message.content)
        return
    await message.channel.send(str(result))
~~~

把辅助 bot 和挑战 bot 放在同一可见频道，启动辅助程序后由用户发送 `$start`。脚本只响应指定 bot 的乘法消息；第 50 轮之后的非算式消息会进入打印分支，其中包含：

~~~text
shellmates{diScORD-1s-TH3-B3st-$oC1AL-M3d1a}
~~~

## 方法总结

限时算术题的核心是可靠地复现消息协议，而不是提高人工计算速度。自动化时要绑定消息来源和频道、避免 eval、处理最终非算式消息，并确保回复是十进制纯数字。赛事原标签“programming”只是题型描述，不是长期知识库中的安全方向。
