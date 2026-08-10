# webpack-engine

## 题目简述

页面加载的是经过高度混淆的 JavaScript，直接阅读打包产物很难定位 flag。关键疏漏是生产构建仍发布了 Webpack source map，浏览器可以据此还原打包前的 Vue 单文件组件。

## 解题过程

打开开发者工具，在 `Sources` 面板中展开站点资源。若 `.map` 文件可访问，Chrome 会显示由 source map 还原出的 `src/views` 等源码目录；本题的目标组件名为 `Fl4g_1s_her3.vue`。

组件模板引用了一个名称近似 `filiiililil4g` 的数据字段，字段值是一串 Base64 文本。不要继续分析混淆后的 bundle，直接复制该字段在当前实例中的完整值并连续解码两次即可：

```javascript
const encoded = prompt("粘贴 Fl4g_1s_her3.vue 中的完整 Base64 字符串").trim();
console.log(atob(atob(encoded)));
```

也可以在终端中完成同样的操作：

```python
import base64

encoded = input("粘贴完整 Base64 字符串：").strip().encode("ascii")
first = base64.b64decode(encoded)
flag = base64.b64decode(first).decode()
print(flag)
```

官方文档中的源码截图在页面右侧截断了编码串，因此这里不臆造不完整的密文或 flag；解题所需的变量位置、编码层数和还原方法均已给出，实际环境中应复制完整字段值。

## 方法总结

前端混淆只能提高阅读成本，不能保护随客户端下发的秘密。排查 Webpack 应用时应同时检查 `.map` 请求、bundle 末尾的 `sourceMappingURL` 和开发者工具中恢复出的源文件树。生产环境不应发布包含敏感源码的 source map，更不能把 flag、密钥或鉴权逻辑放在浏览器可取得的资源里。
