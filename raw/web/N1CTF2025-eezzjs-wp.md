# N1CTF 2025 eezzjs

## 题目简述

题目是一个 Node.js/Express 应用。用户登录后可以向 `/upload` 上传 Base64 编码的文件，首页还会把查询参数 `templ` 交给 `res.render()`。真正的攻击面由两处问题串联而成：应用使用存在长度混淆问题的 `sha.js` 2.4.10 自行实现 JWT 签名，上传接口又只检查扩展名中是否出现 `js`，既没有消除目录穿越，也没有禁止 Node 原生扩展 `.node`。

最终目标不是篡改模板正文，而是先伪造登录令牌，再把恶意原生扩展放入 `node_modules`，借 Express/Node 的模块解析过程加载它。题目源码、官方 PoC 与原生扩展源码共同给出了这条完整链路。

## 解题过程

应用的令牌格式类似 JWT，但签名不是标准的 HMAC。`auth.js` 依次把序列化后的 header、payload 对象和服务端 secret 喂给同一个 SHA-256 实例。其关键逻辑可以概括为：

```javascript
const sha = require('sha.js')('sha256');
sha.update(JSON.stringify(header));
sha.update(payload);
sha.update(secret);
return sha.digest('hex');
```

这里使用的 `sha.js` 版本会信任对象自带的 `length` 属性，并把它加入内部长度计数。payload 中放入负数长度后，可以让计数器回退。官方构造令 payload 的 `length` 为 `-45`：header 已计入的 27 字节与随后加入的 18 字节 secret 正好被抵消，最终内部长度变成 0。散列状态因此等价于计算空消息的 SHA-256，签名不再依赖未知 secret：

```text
SHA256("") = 674dcdbbb09261235ee8efc1999daee725dad0ec314a8d1d80cb11229e7596c1
```

把管理员身份和负长度一起放进 payload，再附上上述十六进制摘要，就能通过认证。该问题对应 `sha.js` 的长度混淆漏洞；上游公告说明 2.4.11 及以前版本受影响，2.4.12 已修复，详见 [GHSA-95m3-7q98-8xr5](https://github.com/browserify/sha.js/security/advisories/GHSA-95m3-7q98-8xr5)。公告本身最重要的信息就是：攻击者可借非标准输入对象操纵内部长度，得到错误但可预测的摘要。

登录后分析上传接口。服务端大致执行：

```javascript
const filename = req.body.filename;
const ext = path.extname(filename).toLowerCase();
if (/js/i.test(ext)) {
    return res.status(403).end();
}
fs.writeFileSync(path.join(uploadDir, filename), data);
```

过滤器只拒绝包含 `js` 的名称，`.node` 不在黑名单内；`path.join` 前也没有校验归一化后的路径是否仍位于上传目录。因此将文件名设为 `../node_modules/exp.node`，即可把一个 ELF 原生扩展写入应用的模块搜索目录。

恶意扩展不必导出正常的模板引擎接口，只需在共享对象被加载时执行构造函数。仓库中的 `exp.cc` 使用了 `cp /flag > /app/uploads/flag.txt`，但这条命令缺少 `cp` 的目标参数，会创建空文件而不能复制 flag；这是官方 PoC 的一个明显笔误。复现时应修正为：

```cpp
__attribute__((constructor))
static void init() {
    system("cp /flag /app/uploads/flag.txt");
}
```

最后访问 `/?templ=1.exp`。Express 发现未知的 `.exp` 视图扩展时，会尝试 `require("exp")` 获取对应模板引擎。Node 的 CommonJS 加载器依次尝试模块目录中的 `.js`、`.json` 和 `.node` 文件，所以会命中刚上传的 `node_modules/exp.node`。原生扩展在 Express 因缺少 `__express` 导出而报错之前已经由 `process.dlopen()` 加载，其构造函数也已经执行。随后读取 `/uploads/flag.txt` 即可得到 flag。Node 的官方[模块文档](https://nodejs.org/api/modules.html)中与本题相关的要点是：找不到精确文件名时会尝试上述扩展名，而 `.node` 会作为编译后的原生插件加载。

完整请求顺序为：

```text
1. 构造 payload.length = -45 的伪造令牌并登录
2. 编译官方 exp.cc，得到 Linux 环境可加载的 exp.node
3. 上传 exp.node，文件名使用 ../node_modules/exp.node
4. 请求 /?templ=1.exp，触发 require("exp")
5. 读取 /uploads/flag.txt
```

## 方法总结

本题的关键是把三个看似独立的实现细节串起来：哈希库信任对象的负 `length`，使自制 JWT 的 secret 被从长度计算中抵消；上传路径未做边界检查，使文件可以进入 `node_modules`；Express 的动态模板扩展又把攻击者控制的扩展名转换成一次 `require()`。单独看每一处都未必立即得到代码执行，组合后却形成“认证绕过—任意位置写文件—原生模块加载”的完整利用链。

复现时需要注意三点：签名字段是题目自定义格式中的十六进制摘要，不要套用标准 JWT 的 Base64URL HMAC；仓库 `poc.js` 内硬编码的摘要只保留了前 56 个十六进制字符，而校验代码要求完整 32 字节摘要，因此应使用正文给出的 64 字符值；原生扩展还必须针对目标容器的 Linux 架构和 Node ABI 编译，否则上传虽然成功，加载阶段仍会失败。
