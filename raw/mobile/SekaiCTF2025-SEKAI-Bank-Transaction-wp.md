# SEKAI Bank - Transaction

## 题目简述

题目提供银行 APK 和官方 PoC APK。目标不是逆向某个本地校验值，而是通过 Android 组件、Intent 与 URI 权限链，修改银行应用私有目录中的延迟交易文件，让管理员账户向攻击者 `nino` 转账 1000000。

关键组件如下：

- `com.sekai.bank.MainActivity`：导出；
- `LogProvider`：`exported=false`，但允许 URI grant；
- `DelayedTransactionReceiver`：不导出，负责执行延迟交易；
- PoC 的 `MainActivity`：导出，用于接收银行应用转交的 Intent。

官方 PoC 使用的 URI 为：

```text
content://com.sekai.bank.logprovider/%2E%2E/files/delayed_transactions
```

## 解题过程

### 1. 用异常路径触发攻击者可控的 fallback

银行 `MainActivity` 会处理 PIN 设置流程。PoC 构造显式指向银行 Activity 的 Intent，并设置：

```text
from_pin_setup = true
context = null
fallback = attackerIntent
```

`setupMainUI()` 继续使用空 `context` 创建 Toast，触发异常。外层异常处理随后启动 `fallback`。

`fallback` 不是普通无权限 Intent，而是 PoC 预先构造、指向攻击者导出 Activity 的 Intent，并设置目标 content URI 以及：

```text
FLAG_GRANT_READ_URI_PERMISSION
FLAG_GRANT_WRITE_URI_PERMISSION
FLAG_GRANT_PERSISTABLE_URI_PERMISSION
```

由于真正执行 `startActivity(fallback)` 的是银行应用，URI 权限由拥有 `LogProvider` 的银行身份授予攻击者，形成 confused deputy。`LogProvider` 即使 `exported=false`，攻击者仍能凭精确 URI grant 访问。

### 2. 用编码路径绕过目录穿越检查

Provider 先对原始 URI 字符串执行：

```java
if (uri.toString().contains("..")) {
    throw ...
}
```

`%2E%2E` 中没有字面量 `..`，检查通过。之后 `uri.getPath()` 返回解码后的路径，文件构造近似为：

```java
new File(getContext().getCacheDir(), decodedPath)
```

于是路径从：

```text
cache/../files/delayed_transactions
```

落到应用私有的 `files/delayed_transactions`。`openFile` 又固定以可读写模式打开文件描述符，使这次目录穿越同时获得读写能力。

### 3. 两阶段获取精确文件权限

延迟交易目录下的 JSON 文件名是动态的。PoC 因此分两阶段运行：

1. 第一阶段请求目录 URI grant，查询目录内容；
2. 找到扩展名为 `.json` 的条目；
3. 第二阶段再次走 fallback，为具体文件 URI 获取 grant；
4. 读取、修改并覆盖该 JSON。

修改的字段为：

```json
{
  "amount": 1000000,
  "toUsername": "nino"
}
```

PoC 不必伪造管理员 PIN。延迟交易文件原本已经保存了合法的 PIN 和消息字段，只改收款人及金额即可。

### 4. 等待银行应用执行延迟交易

`DelayedTransactionManager` 后续从私有 `files` 目录读取 JSON，按存储内容构造：

```text
SendMoneyRequest(toUsername, amount, message, pin)
```

请求由银行应用自己的已认证 API 客户端发送。服务器因此看到的是管理员身份与原有有效 PIN，只是金额和收款人已被本地文件篡改。交易执行后，攻击者登录 `nino` 查看交易历史即可取得 flag。

正式 API 部署配置中，这一百万转账条件对应第二阶段 flag：

```text
SEKAI{so-many-trouble-just-to-steal-a-million}
```

## 方法总结

这条链需要四个条件同时成立：

1. 导出 Activity 在异常路径中启动攻击者提供的 Intent；
2. 启动者拥有 Provider，因而能替外部应用授予 URI 权限；
3. Provider 在解码前检查路径，却在解码后使用路径；
4. 延迟任务信任私有 JSON 中的敏感业务字段。

`exported=false` 不是绝对隔离，只要组件支持 URI grant，就必须审计所有能够代表应用发放权限的 Intent 流。路径校验则应在规范化后的文件对象上完成，并确认最终 canonical path 仍位于允许根目录内。
