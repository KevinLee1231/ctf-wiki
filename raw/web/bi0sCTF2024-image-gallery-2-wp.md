# bi0sCTF 2024 - Image Gallery 2

## 题目简述

第二版图库改为 Go 应用并在前面加入 Nginx。管理员机器人先创建自己的 UUID 会话，把 flag 内容上传为 `flag.png`，随后访问选手提交的同站 URL。页面通过 `/files.js` 获取当前会话的文件列表和 `sid`，再由 `/main.js` 生成图片元素。

题目的目标不是直接注入 JavaScript，而是组合 Nginx 的 4096 字节 slice 缓存、脚本 SRI 完整性校验、DOM Clobbering 与浏览器图片缓存探测，逐段恢复管理员 UUID。得到该目录名后，即可直接读取 `/static/<admin-sid>/flag.png`。

## 解题过程

### 识别动态脚本中的秘密

每次请求 `/files.js`，后端都会根据 Cookie 对应目录重新生成脚本：

```javascript
if (top.location.origin === 'APP_ORIGIN')
fileNames = JSON.parse(atob(decodeURIComponent('<base64-list>'))),
id = '<sid>';
```

管理员上传 flag 后，列表中会出现 `flag.png`，而脚本末尾的 `id` 是管理员 UUID。`/main.js` 随后拼接：

```javascript
imgElement.src = `/static/${id}/${fileName}`;
```

静态文件路由只要求请求 Cookie 看起来像 UUID，并没有要求 URL 中的目录与 Cookie 相同。因此真正需要泄漏的是管理员 `sid`；恢复它后，攻击者自己的合法 Cookie 也能请求管理员目录中的 `flag.png`。

### 把不同版本的 `files.js` 拼进同一缓存对象

Nginx 对 `.js` 等后缀启用了：

```nginx
slice 4096;
proxy_cache_key $host$uri$is_args$args$slice_range$cookie_sid;
proxy_set_header Range $slice_range;
```

缓存键包含查询串、Cookie 和 slice 范围，但不包含后端文件版本。攻击页先上传大量长文件名，把 `files.js` 推过 4096 字节边界，并为多个查询键缓存 `bytes=0-4095`。随后删除或替换某个特定长度的文件名，令脚本长度和边界附近内容改变，再为同一查询键缓存第二个 slice。

之后浏览器请求完整 `/files.js?<key>` 时，Nginx 可把来自两个不同文件版本的缓存分片组合起来。通过精确选择临时文件名长度，可以让拼接后的脚本在 `id = '<sid>'` 附近截断，并使每个查询键对应 UUID 的不同位置。官方 exploit 使用 18 个查询键覆盖 UUID 的 32 个十六进制字符和 4 个连字符。

### 用 SRI 把候选字符串变成执行与否

对某个待恢复位置，枚举 256 个十六进制字节候选；连字符位置只需枚举 16 个半字节。对每个候选，离线构造预期的截断 `files.js` 全文并计算：

```python
digest = hashlib.sha256(candidate_script.encode()).digest()
integrity = "sha256-" + base64.b64encode(digest).decode()
```

把候选放进：

```html
<script integrity="sha256-..." src="/files.js?KEY"></script>
```

浏览器只有在实际缓存脚本的 SHA-256 与候选 SRI 完全相等时才执行该脚本。SRI 因而成为一个严格的相等性 oracle，无需直接读取管理员脚本响应。

### DOM Clobbering 与图片缓存回传

候选页面同时放置重复 `id` 的锚点：

```html
<a id="id" href="abc:placeholder"></a>
<a id="fileNames" href="x:../../../../../static/KNOWN_IMAGE?candidate"></a>
<a id="fileNames" href="x:placeholder"></a>
```

命名元素会把全局 `id` 与 `fileNames` 替换为锚点或锚点集合。随后加载合法 `/main.js` 时，模板字符串会对这些 DOM 对象做字符串转换，路径中的 `../` 再把请求归一化到一张攻击者可控的已知图片，并附带候选查询串。

只有 SRI 命中的候选会执行动态脚本并继续触发相应图片请求，所以该候选图片会进入管理员浏览器缓存。攻击页再逐个创建相同 URL 的 `Image`，利用 `img.complete` 或加载时间区分缓存命中，即可确定正确候选。每轮更新已知后缀并切换查询键，最终拼出完整管理员 UUID。

最后请求：

```text
/static/<recovered-admin-sid>/flag.png
```

文件虽然扩展名是 PNG，内容实际就是 flag 文本。题目要求把挑战源设为安全上下文，是为了使所依赖的浏览器缓存行为在 HTTP 部署下保持一致。

## 方法总结

本题把多个单点看来影响有限的行为串成跨会话泄漏：slice 缓存混合不同版本的动态脚本，SRI 把未知脚本内容转成候选相等性判断，DOM Clobbering 让固定的 `/main.js` 发出候选相关请求，浏览器图片缓存再把执行结果带回攻击者。最终泄漏的是管理员目录 UUID，而非直接读取 flag 内容；静态路由没有绑定 Cookie 与路径，才使该 UUID 足以完成最后读取。
