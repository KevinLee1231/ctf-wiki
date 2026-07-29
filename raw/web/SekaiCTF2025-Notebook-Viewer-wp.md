# Notebook Viewer

## 题目简述

管理员先打开：

```text
/?note=<FLAG>
```

主页面对 flag 的每个字符创建一个 iframe：

```text
https://nbv-<index>-<codepoint>.chals.sekai.team/
```

iframe 被放进 closed Shadow DOM，攻击页不能直接枚举；但字符值已经编码进跨源 iframe 的 origin。各子域响应设置 `Origin-Agent-Cluster: ?1`，使不同 origin 使用独立 agent cluster。攻击者可以通过候选 origin 的 JavaScript 堆占用判断某个 origin 是否已经被隐藏 iframe 使用。

## 解题过程

### 1. 取得候选 origin 上的脚本执行

子域页面读取 `note` 参数后直接：

```javascript
txt.innerHTML = note;
```

因此可加载：

```html
<svg onload="...">
```

候选 iframe URL 为：

```text
https://nbv-<idx>-<candidateCode>.chals.sekai.team/frame.html?note=<XSS>
```

注入脚本读取：

```javascript
performance.memory.usedJSHeapSize
```

再通过 `top.postMessage` 把数值和候选字符发回攻击页。

### 2. 为什么堆大小能泄露字符

如果候选字符正确，攻击 iframe 与管理员第一页中对应字符 iframe 的 scheme、host、port 完全相同。它们被放入同一个 origin agent cluster，测得的 JS heap 已包含隐藏页面的文档和运行时对象，数值较大。

若候选错误，该子域 origin 此前没有页面，创建的是较干净的 agent cluster，堆占用较小。

closed Shadow DOM 只阻止 DOM API 取得内部节点，不会隐藏 iframe 已建立的网络 origin 和浏览器进程状态，因此不能阻断此侧信道。

### 3. 建立基线并逐字符枚举

flag 已知以 `SEKAI{` 开头。先测试索引 0 的字符 `S`，以正确 origin 的堆占用建立基线：

```javascript
threshold = baseline - baseline / 10;
```

之后对每个索引按允许字符集分批测试 4 个候选。每个候选：

1. 创建带 XSS 的 iframe；
2. 等待其回传 heap；
3. 删除 iframe；
4. 选择高于阈值或本批最大值的字符。

重复到恢复 `}`。分批创建而不是同时加载整个字符集，可以降低各 agent cluster 初始化噪声和内存压力。

### 4. 外带进度

官方 solver 每恢复一个前缀就用 `navigator.sendBeacon` 发送进度。实际部署时只需把示例收集地址换成自己的 HTTPS 接收端；题目管理员等待 15 秒，因此候选批次、超时和字符集需要保持紧凑。

## 方法总结

本题不是传统 XS-Leak 的状态码或加载时间，而是 origin 隔离下的 heap 占用侧信道：

```text
字符编码进子域
→ 正确候选与隐藏 iframe 同源
→ 共享 agent cluster
→ usedJSHeapSize 偏高
```

敏感值不应出现在攻击者可枚举的 host、URL 或资源 origin 中。closed Shadow DOM 不是保密边界；它只封装 DOM 访问，无法隐藏网络请求、进程分组和内存等浏览器可观察状态。
