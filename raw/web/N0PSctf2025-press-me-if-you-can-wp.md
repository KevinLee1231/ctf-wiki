# Press Me If You Can

## 题目简述

页面上只有一个会躲避鼠标的按钮。JavaScript 在光标靠近时移动按钮，并在每次 `mousemove` 事件中把它设为禁用；服务端收到正常的表单 POST 后，会把 flag 写入浏览器控制台。题目考查的是浏览器端状态可由用户自行修改，而不是寻找复杂的服务端漏洞。

## 解题过程

### 分析按钮为何无法手动点击

按钮本身是普通表单控件：

```html
<form method="post">
    <button type="submit" name="submit">Press Me</button>
</form>
```

阻碍点击的逻辑全部位于客户端。鼠标移动时，脚本根据光标与按钮的距离更新位置，并执行：

```javascript
btn.disabled = true;
```

DOM、事件处理器和 `disabled` 属性都不构成安全边界。打开开发者工具的 Console，在同一段同步代码中恢复按钮并触发点击即可：

```javascript
const button = document.querySelector("button");
button.disabled = false;
button.click();
```

`click()` 会提交原表单。浏览器发出的同源请求带有正常的 `Referer` 和浏览器 User-Agent，可以通过服务端的弱检查：

```php
if (
    !isset($_SERVER['HTTP_REFERER'])
    || str_starts_with($_SERVER['HTTP_USER_AGENT'], "curl")
) {
    die("I saw that you did not press the button");
}
```

服务器返回一段 `console.log(...)` 脚本，因此 flag 不会直接显示在页面正文中。查看 Console 可见：

```text
N0PS{W3l1_i7_w4S_Ju5T_F0r_Fun}
```

### 直接复现服务端请求

服务端并不能证明用户真的点击过按钮：它只检查 `Referer` 是否存在，以及 User-Agent 是否以 `curl` 开头。手工构造同样可以通过：

```bash
curl 'http://target/' \
  -H 'Referer: http://target/' \
  -H 'User-Agent: Mozilla/5.0' \
  --data 'submit=1'
```

响应 HTML 中同样包含完整 flag。

## 方法总结

本题的关键是区分交互效果和访问控制。按钮移动、禁用以及控制台显示都只发生在攻击者完全可控的浏览器里；修改 DOM 或直接构造 POST 都能绕过。原官方题解中的开发者工具截图只是操作记录，没有不可替代的视觉证据，因此将其转写为命令和源码片段，不再保留图片。
