# CSEC Invasion 3

## 题目简述

题面中的 “manifest” 与 “mash” 提示 Progressive Web App 的清单文件。网站把自定义计算字段留在公开的 `manifest.json` 中。

## 解题过程

先从 HTML 的 `<link rel="manifest">` 确认清单路径，或直接访问：

```text
/manifest.json
```

JSON 中的 `calc` 字段包含：

```text
UMDCTF-{w3_d1d_th3_m@th}
```

浏览器会自动下载 Web App Manifest 以获取应用名称、图标和启动配置，因此其中的所有额外字段同样对用户公开。

## 方法总结

除 HTML 和 JavaScript 外，PWA 清单、source map、预缓存列表等构建元数据也属于客户端攻击面。配置文件中不应存放秘密，即使应用代码从未主动读取该字段。
