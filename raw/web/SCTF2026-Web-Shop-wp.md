# Web Shop

## 题目简述

新用户有 50 金币，通过木鱼最多再得 10 金币，正好可以买 `Support Debug Bundle`。下载包公开了 support ticket 的 HMAC 算法，但密钥来自环境变量 `SHOP_SUPPORT_SEED`。聊天后端会用 LangChain `loads()` 恢复用户可控 metadata，攻击者可注入 `type=secret` 对象读取该环境变量，伪造当日票据登录为 `support_admin`。

管理员可进入 Rule Lab。沙箱用 AST 黑名单禁止 `gi_frame`、`f_locals` 等属性，但允许运行时拼接字符串并调用 `str.format`；format field 自带属性和 item traversal，可间接读取生成器 frame 中的 `shipment_manifest`，其内容即部署 flag。

## 解题过程

### 1. 购买并分析 Support Debug Bundle

注册后连续调用 10 次 `/api/woodfish/knock`，金币从 50 变为 60。购买调试包并下载 `support_ticket.py`，可见票据算法：

```python
today = datetime.now(timezone.utc).strftime('%Y%m%d')
message = f"support-login:{user['id']}:{user['username']}:{today}"
ticket = hmac.new(
    SHOP_SUPPORT_SEED.encode(),
    message.encode(),
    hashlib.sha256
).hexdigest()[:12]
```

因此只缺环境变量 `SHOP_SUPPORT_SEED`。日期使用 UTC，临近本地午夜时不能用本地日期代替。

### 2. 通过 LangChain secret 泄漏种子

聊天 presence 包表明 metadata 使用 LangChain serialized object 格式。向 `/api/chat/messages` 写入：

```json
{
  "content": "hello",
  "metadata": {
    "probe": {
      "lc": 1,
      "type": "secret",
      "id": ["SHOP_SUPPORT_SEED"]
    }
  }
}
```

消息入库时结构被保留；再次 GET 聊天历史时，后端对 metadata 调用 `loads()`，LangChain secret loader 从环境变量解析该 id，响应中的 `metadata.probe` 变成真实 seed。历史接口只返回系统消息和当前用户自己的消息，故应读取刚写入的那条记录，而不是等待别人的 payload。

### 3. 伪造票据并提权

按下载脚本的原始用户 id、username 和当前 UTC 日期计算 HMAC-SHA256 前 12 个 hex 字符。把结果发给 Bot：

```text
/login <ticket>
```

成功后用 `/api/auth/me` 确认角色已变为 `support_admin`，再访问 `/api/rules/run`。如果只看 Bot 的自然语言回复而不验证角色，容易把失败票据或日期偏差带入沙箱阶段。

### 4. 绕过 Rule Lab AST 黑名单

Rule Lab 提供 `iter_preview_items()`。其生成器函数先把私有发货文件读入局部变量 `shipment_manifest`，随后才 yield 预览条目。直接写 `g.gi_frame.f_locals` 会被 AST 拒绝，但黑名单只检查源码中完整属性名和字符串常量。

提交：

```python
g = iter_preview_items()
next(g)
field = "{0.gi_" + "frame.f_" + "locals[shipment_manifest]}"
result = field.format(g)
```

`next(g)` 让局部变量已经初始化且 frame 仍挂起。运行时拼接得到：

```text
{0.gi_frame.f_locals[shipment_manifest]}
```

`str.format` 的 field parser 会执行属性访问和字典索引，等价于读取 `g.gi_frame.f_locals['shipment_manifest']`，却没有在 AST 中出现被禁的完整 token。接口返回的 `result` 即为实际 flag。

## 方法总结

本题连续跨越三类边界：业务金币提供调试工件，反序列化器把 metadata 当成有语义对象，字符串格式化器又在“数据字符串”中执行属性遍历。修复不能只补最后一个 payload：LangChain `loads()` 应禁用用户可控 secret 类型或使用显式 allowlist；support ticket 不应把签发材料泄给客户端；Rule Lab 应采用能力受限解释器/独立进程，而不是 AST 黑名单。题解中的 flag 不写死，最终以当前部署 `shipment_manifest` 返回值为准。
