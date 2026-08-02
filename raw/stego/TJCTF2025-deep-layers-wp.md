# deep-layers

## 题目简述

附件是一张可正常解析的 1×1 PNG，但文件在 PNG 结束位置之后还拼接了一个加密 ZIP。ZIP 中包含 `secret.gz`，而解压密码以 Base64 形式存放在 PNG 的 `Password` iTXt 元数据中。题目的“层”依次是 PNG 元数据、尾随 ZIP 和 Gzip 数据。

## 解题过程

不能只按图片像素检查。PNG 解码器会在 `IEND` 后停止，而 ZIP 工具可以从后半段的 `PK\x03\x04` 本地文件头继续解析，所以同一字节流同时承担两种格式。

完整提取脚本如下：

```python
import base64
import gzip
import zipfile
from io import BytesIO
from PIL import Image

path = "chall.png"

with Image.open(path) as image:
    encoded_password = image.info["Password"]
password = base64.b64decode(encoded_password)

with open(path, "rb") as f:
    raw = f.read()

zip_offset = raw.find(b"PK\x03\x04")
if zip_offset < 0:
    raise RuntimeError("ZIP local header not found")

with zipfile.ZipFile(BytesIO(raw[zip_offset:])) as archive:
    compressed = archive.read("secret.gz", pwd=password)

print(gzip.decompress(compressed).decode().strip())
```

仓库样本运行结果为：

```text
tjctf{p0lygl0t_r3bb1t_h0l3}
```

1×1 图片本身没有可用视觉内容，因此不保留无意义的黑色缩略图；真正需要长期保存的是各层格式、密码位置和提取顺序。

## 方法总结

- 核心技巧：利用文件格式解析边界，从 PNG 元数据取密码并从尾随 ZIP/Gzip 中逐层提取载荷。
- 识别信号：文件尺寸明显大于图像内容所需、PNG 尾部后出现 `PK` 魔数、元数据存在可疑 Base64 字段。
- 复用要点：polyglot 题应同时检查格式内元数据和规范结束标记后的尾随字节；每拆一层都先确认魔数和真实格式。
