# GreyCTF 2025 English or Spanish Writeup

## 题目简述

应用通过 `/translations/:lang/:prop` 加载语言模块，再从模块对象中读取指定属性。后端同时导出了一个文件读取代理，但正常页面不会调用它。目标是把可控的 `lang` 与 `prop` 串联起来，让动态 import 加载 `importer.js` 自身，并通过代理读取 flag 文件。

## 解题过程

语言加载逻辑为：

```javascript
const { default: language } = await import(`./languages/${lang}/lang.js`);
return propertyAccessor(language);
```

`propertyAccessor` 会把一个属性名按点号拆分后逐层访问。模块还导出：

```javascript
importFile: new Proxy({}, {
  get: (_, file) => fs.readFileSync(file).toString(),
})
```

利用请求为：

```text
/translations/..%2Fimporter.js%23/importFile.flag
```

Express 解码后，`lang` 成为 `../importer.js#`。拼入 import 路径得到近似如下的模块标识符：

```text
./languages/../importer.js#/lang.js
```

其中 `#` 后的内容被当作 URL fragment，不参与实际文件定位，因此加载的是 `./importer.js`。该 CommonJS 模块经动态 import 后出现在 `default` 中。第二个路由参数 `importFile.flag` 被属性访问器拆成两段：先取得 `importFile` 代理，再访问 `flag` 属性；代理的 `get` 钩子随即执行 `readFileSync('flag')`。

响应是 JSON 字符串，去掉 JSON 引号和文件末尾换行即可得到：

```text
grey{why_ju5t_why_j5_1mp0rts}
```

## 方法总结

利用链依赖三个语义叠加：路由参数允许编码斜杠、动态 import 把 `#` 解释为 fragment、点分属性访问最终落到文件代理。单独看任一处都不一定能读文件，但它们组合后使攻击者既能选择模块，又能选择代理和文件名。审计动态模块加载时，应对插值内容做白名单映射，而不是直接拼路径；同时不应把任意文件读取能力挂在可遍历的导出对象上。
