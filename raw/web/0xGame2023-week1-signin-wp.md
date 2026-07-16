# signin

## 题目简述

页面由 Vite 打包，生产环境 JavaScript 虽然经过压缩，但仍公开了 source map。映射文件保存了打包产物与原始源码之间的对应关系，并可直接包含 `sourcesContent`；题目的 flag 就位于原始的 `/src/main.js` 中。

## 解题过程

打开开发者工具的“源代码/来源”面板，可以在 source map 还原出的目录树中找到 `/src/main.js`。该文件中的有效内容为：

```javascript
let flag = '0xGame{c788fa80-2184-429f-b410-48cb8e2de0ff}';
```

也可以直接检查页面加载的 `/assets/index-33309f51.js`。文件末尾给出了映射文件名：

```javascript
//# sourceMappingURL=index-33309f51.js.map
```

请求对应的 `/assets/index-33309f51.js.map`，在返回的 JSON 中搜索 `src/main.js` 或 `0xGame`。映射文件的 `sources` 用于标识原文件，`sourcesContent` 则保存原始内容，因此无需逆向压缩后的 bundle 即可读出 flag。

## 方法总结

前端构建产物暴露 `.map` 文件会泄露原始目录、文件名、注释和硬编码敏感信息。遇到 Vite、Webpack 等打包页面时，应检查脚本末尾的 `sourceMappingURL`，并审查映射文件中的 `sourcesContent`；生产部署应禁用公开 source map 或将其置于不可访问的位置。
