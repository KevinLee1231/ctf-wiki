# DownUnderCTF 2021 - Jasons Proxy

## 题目简述

图片加载接口使用自制 JSON 解析器，并把 URL 转交给另一层 CDN 代理。重复的 `img` 键可以让第二个值跳过第一层安全检查；在第二层再组合多重 URL 编码与路径规范化差异，就能把白名单图片路径变成内部 `/admin/flag`，形成 SSRF。

## 解题过程

### 用重复键绕过自制 JSON 检查

`JSON.parse` 手动按逗号、冒号切分输入，并用 `checked` 列表避免重复检查：

```python
def _security(self, key, value):
    if key in self.checked:
        return False
    # 检查危险字符，并对 img 做一次 Base64 解码检查
    ...
    self.checked.append(key)
```

但解析器随后仍会处理第二个同名键，最终字典保留后一个值。于是提交两个 `img`：第一个是无害的 `aHR0`，Base64 解码为 `htt`，通过检查并把键记为已检查；第二个恶意值不再进入安全逻辑，却仍会被 Base64 解码并覆盖第一个值。

完整请求体为：

```json
{
  "img": "aHR0",
  "img": "aHR0cDovLzEyNy4wLjAuMS9zdGF0aWMvaW1hZ2VzLyUyZSUyNTJlLyUyZSUyNTJlLyUyNTYxZG1pbi9mbCUyNTYxZw=="
}
```

第二段解码后是：

```text
http://127.0.0.1/static/images/%2e%252e/%2e%252e/%2561dmin/fl%2561g
```

它满足前端代码要求的固定前缀 `http://127.0.0.1/static/images/`。

### 利用代理各阶段解码顺序

CDN 代理的 WAF 先执行一次 `unquote`，随后检查是否以白名单开头、是否含 `admin` 或 `flag`，再做字符串替换；最后使用 `abspath` 规范化路径、再次 `unquote`，并与白名单基址执行 `urljoin`。

恶意路径在各阶段的含义如下：

1. `%2e%252e` 第一次解码为 `.%2e`，尚不是字面量 `..`，不会命中 `../` 替换；
2. `%2561dmin` 与 `fl%2561g` 第一次解码为 `%61dmin` 和 `fl%61g`，不会命中黑名单 `admin`、`flag`；
3. 后续再次解码时，`.%2e/.%2e` 变成 `../../`，两个 `%61` 变成字母 `a`；
4. 规范化并与基址合并后，实际目标成为 `http://127.0.0.1/admin/flag`。

该端点只允许来源地址为 `127.0.0.1`，恰好由内部代理满足。应用把响应内容再次 Base64 编码，返回类似：

```json
{
  "imagedata": "RFVDVEZ7ZDB1YmwzX2pzMG5fZDB1YmwzX1VSSV9yMXBfajRzMG41X3A0dGhfdzF0aF9iMWdfaDR4eH0="
}
```

解码 `imagedata` 得到：

```text
DUCTF{d0ubl3_js0n_d0ubl3_URI_r1p_j4s0n5_p4th_w1th_b1g_h4xx}
```

## 方法总结

本题的两次绕过来自不同解析层：自制 JSON 对重复键只检查第一次却使用最后一次；URL WAF 又在解码、黑名单检查和路径规范化之间采用了不一致的表示。安全实现应使用标准 JSON 解析器并拒绝重复敏感键，对 URL 只在完全解析、递归解码受控且规范化完成后做一次基于目标地址和路径的允许列表检查，同时阻止代理访问回环与内部管理端点。
