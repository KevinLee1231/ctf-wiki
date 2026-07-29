# rednote

## 题目简述

管理员注册随机账户，并创建标题为 `flag`、正文为 flag 的笔记，然后在同一浏览器上下文访问攻击页。`/search?query=` 会在当前用户笔记中返回第一条标题或正文包含查询串的记录。

笔记正文经 DOMPurify 清洗，但允许 `<style>`。目标页面还设置：

```text
Document-Policy: force-load-at-top
```

官方解法利用 Chromium 中 `@starting-style` 与 `::first-letter` 组合触发 renderer crash，把“搜索返回 flag 笔记还是攻击者笔记”转成窗口是否成功提交跨源导航的 oracle。

## 解题过程

### 1. 为每个猜测创建崩溃笔记

对当前已知前缀 `KNOWN` 和候选字符 `c`，令：

```text
q = KNOWN + c
```

通过跨站表单在管理员账户中创建：

- title：`q`
- note：崩溃 CSS

崩溃正文为：

```html
hihi
<style>
  @starting-style {
    *::first-letter {
      color: red;
    }
  }
</style>
```

`style` 在应用的 `TAGS` 白名单内，因此 DOMPurify 保留它。官方攻击页使用带 `allow-forms allow-scripts allow-popups` 的 sandbox iframe 和命名弹窗提交表单；管理员会话刚创建，浏览器的 Cookie 行为允许该顶层 POST 进入已登录会话。

### 2. 利用搜索的第一匹配

管理员已有 flag 笔记，而且它早于攻击者新建的探测笔记。

若 `q` 是正确 flag 前缀：

```text
flag note.note.includes(q) == true
```

`find()` 首先返回正常 flag 笔记，不加载崩溃 CSS。

若 `q` 错误：

```text
flag note 不匹配
探测 note.title == q
```

搜索返回攻击者笔记，页面插入崩溃 CSS，renderer 终止。

### 3. 把 renderer crash 转成布尔值

攻击页打开搜索结果窗口，等待页面加载后执行：

```javascript
w.location = "about:blank";
w.location = url + "#" + Math.random();
```

短暂等待后尝试读取：

```javascript
w.location.href
```

- 正确前缀：正常目标页已提交为跨源页面，读取 `location.href` 抛出异常，oracle 返回真；
- 错误前缀：目标 renderer 在提交过程中崩溃，窗口仍停留在攻击者可读的 `about:blank`，读取成功，oracle 返回假。

随机 hash 防止复用旧导航状态。

### 4. 逐字符恢复

题目给出 flag 字符集只含小写字母和下划线。官方顺序为：

```text
}_abcdefghijklmnopqrstuvwxyz
```

从 `SEKAI{` 开始逐字符查询，命中后重置字符集，直到末尾 `}`。应用每 4 秒最多 4 次请求，攻击脚本在创建笔记和查询之间显式等待，以避免 rate limit 干扰 crash oracle。

## 方法总结

本题把三个看似无关的功能连接起来：

```text
搜索首条匹配
→ 允许 style 的富文本
→ 浏览器 renderer crash
→ 跨源导航提交状态
```

DOMPurify 只保证白名单语法，并不能保证允许的 CSS 不触发浏览器漏洞。敏感搜索不应向跨源上下文暴露可区分的渲染行为；管理员 bot 也应隔离会话、禁用跨站创建内容，并限制可访问的外部页面。
