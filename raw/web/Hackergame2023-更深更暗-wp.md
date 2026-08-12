# 更深更暗

## 题目简述

页面会不断向下增长，要求在“无底洞”深处寻找 flag。滚动只是视觉干扰：flag 已由前端根据当前用户 token 生成并写入 DOM，随后脚本再持续插入内容，把它从视野中推远。

## 解题过程

前端的关键逻辑可整理为：

```javascript
function getFlag(token) {
  const hash = CryptoJS.SHA256(`dEEper_@nd_d@rKer_${token}`).toString();
  return `flag{T1t@n_${hash.slice(0, 32)}}`;
}
```

页面从查询参数或 `localStorage` 取得 token，向后端发送一次 `POST /` 验证 token，然后在本地调用 `getFlag(token)`。约 500 毫秒后，脚本把结果写入 `#titan` 元素；同时点击回到顶部，并借助 `IntersectionObserver` 持续在 flag 之前插入新行，于是正常滚动看起来永远到不了目标。

最直接的解法是打开开发者工具的 Elements（元素）面板，在 DOM 中搜索：

```text
flag{
```

搜索会直接命中 `#titan` 中已经生成的文本，不受页面滚动长度影响。另一种方法是在开发者工具中暂停 JavaScript，再滚动到页面底部；暂停后观察器无法继续插入新元素，也能看到 flag。

如果需要独立复算，可从自己的页面会话中读取 token，按源码中的固定前缀拼接，计算 SHA-256，并取十六进制摘要前 32 个字符：

```python
import hashlib

token = "<自己的 token>"
digest = hashlib.sha256(f"dEEper_@nd_d@rKer_{token}".encode()).hexdigest()
print(f"flag{{T1t@n_{digest[:32]}}}")
```

这里不能使用别人的 token，因为 flag 与当前 token 绑定；示例中的占位符也不是可提交凭据。

## 方法总结

本题的决定性机制是浏览器端 DOM 与脚本行为，因此归类为 Web。无限滚动不代表数据尚未产生：应先检查页面源码、运行时 DOM、网络请求和定时器，判断目标是被延迟加载、被视觉遮挡，还是已在本地生成。本题中停止 DOM 增长或直接搜索 DOM 都比继续滚动可靠；源码还给出了可复算的完整 flag 派生算法。
