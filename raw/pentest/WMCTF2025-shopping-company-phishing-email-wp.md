# Shopping company & phishing email

## 题目简述

这篇 WP 覆盖两个 Misc 题：`Shopping company` 和 `phishing email`。原题解外链是出题人博客：<https://www.snowywar.top/4622.html>，博客的关键信息不是“看链接即可”，而是两道题都在模拟真实攻防场景：前者是客服系统、AI 工具调用、内网服务和 Web 渗透组成的攻击链；后者是带恶意 SVG 附件的钓鱼邮件。

`Shopping company` 的入口是一个电商客服系统。用户侧上传文件并发消息后，Node.js 服务会通过 webhook 把消息、文件 URL 和历史上下文交给 AI 客服；AI 客服使用 LangChain tool 调用图片分析和压缩包分析工具。漏洞点在压缩包分析工具：解压后遍历文件时，如果文件头被识别为可执行文件，会 `chmod +x` 后直接执行该文件。

附件 `docker-compose.yml` 明确了三层网络：`part1` 是唯一外网入口，暴露 `3000`，地址 `172.20.0.10`；`ai_part` 同时连生产网和办公网，地址为 `172.20.0.20`、`192.168.100.10`；`webui_part`、`redis`、`part2` 位于公司内网，分别是 `10.0.1.10`、`10.0.1.20`、`10.0.1.30`。因此第一段 RCE 落在 `ai_part` 后，继续打内网服务才是完整链路。

`phishing email` 的附件是一封 `.eml` 邮件，正文伪装成逾期发票催收，附件名为 `Invoice_INV-619682.svg`。题目参考了 Cloudflare 关于 SVG 钓鱼的报告：SVG 是 XML 文档，不只是普通图片，可以包含 JavaScript、操作 DOM、自动跳转、渲染自包含钓鱼页或在弱 CSP 页面里注入脚本。报告链接保留为来源：<https://www.cloudflare.com/cloudforce-one/research/svgs-the-hackers-canvas/>。

## 解题过程

### Shopping company

客服系统的用户端、客服端和文件上传逻辑都在 `part1`。用户发送消息后才触发 webhook，客服回复不会触发；这意味着攻击动作需要伪装成普通用户咨询或文件投递。AI 侧 `/ai-webhook` 要求 `Authorization: Bearer default-token`，收到 `message_type=file` 且文件扩展名为 `zip`、`rar`、`7z` 时，会提示模型调用 `extract_and_analyze_archive`。

危险逻辑集中在 `tools.py` 的压缩包分析：

```python
def is_executable_binary(file_path: Path) -> bool:
    with open(file_path, 'rb') as f:
        header = f.read(4)
        return header in {
            b'\x7fELF',
            b'MZ\x90\x00',
            b'PK\x03\x04',
            b'\xca\xfe\xba\xbe',
            b'\xcf\xfa\xed\xfe',
        }

elif is_executable_binary(file_path):
    os.chmod(file_path, 0o755)
    output = os.popen(str(file_path)).read()
```

这里不是按后缀判断，而是按文件头判断。直接上传 ELF 时，客服可能不“打开”；把可执行文件打包进 zip，再通过 prompt 注入诱导 AI 分析压缩包，就会进入解压、识别、执行这条链。实际利用时将回连程序或执行命令的 ELF 放入 zip，发送给客服并在聊天内容中强调“请解压并检查附件内容”，AI 工具调用后即可在客服容器内执行。

拿到入口机器后，第二段目标是内网里的 Open WebUI。附件 `webui_part/Dockerfile` 使用 `ghcr.io/open-webui/open-webui:0.6.1`，默认创建 `admin@admin.com / 123456`，并把 `wmctf{AI_is_the_future}` 写入 `/flag`。关键路径是先探测 `10.0.1.10:8080`，登录 Open WebUI；再组合两处 N-day：上传 gguf 模型导致任意文件覆盖，用它覆盖 `main.py`；随后向 `/api/v1/utils/code/format` 传入大量数据触发服务重启。覆盖文件只有在进程重启后生效，因此“文件覆盖 + 重启触发”是完整利用链。恶意 `main.py` 生效后回连 shell，读取 Open WebUI 容器内 flag。

第三段是普通 Web 渗透服务。附件 `part2/main.go`、`part2/SOLUTION.md` 和 `part2/exploit.py` 都指向数据分析模块：登录 `admin/admin` 后创建分析报告，报告数据会序列化进 Redis，查看报告时再反序列化。当前附件源码中稳定利用路径是 `template.commands`，`createAnalyticsHandler` 遇到 `template` 字段时会把整段 JSON base64 后写入 Redis，`viewAnalyticsHandler` 取出后调用 `DataProcessor.GobDecode(data)`，再由 `commands` 数组触发 `exec.Command("sh","-c",cmdStr).Run()`。

```json
{
  "data": {
    "type": "system_analysis",
    "period": "monthly",
    "metrics": ["cpu", "memory", "disk"]
  },
  "template": {
    "template_type": "dynamic",
    "fields": ["performance", "usage"],
    "commands": ["id"]
  }
}
```

这里的核心是让后端把用户输入带入报告模板，随后在 Redis 取出并反序列化报告时触发命令执行。验证方式可以先写入 Web 静态目录：

```bash
echo RCE_SUCCESS > /app/static/rce_test.txt
```

然后访问静态文件确认命令确实执行。读取 flag 时同理，把 `/flag.txt` 写到 Web 可访问位置，或者用 HTTP/DNS 外带。

附件 `part2/flag.txt` 中的结果为：

```text
wmctf{G0_D3s3r14l1z4t10n_R3d1s_RC3_Ch4ll3ng3_2025}
```

### phishing email

附件 zip 中只有一封邮件 `phishing_email_1_20250727_202246.eml`。邮件头和正文伪装得像企业催收：

```text
From: Sarah Allen <sarah_allen@servicenow.com>
To: Patricia Robinson <probinson@spotify.com>
Subject: OVERDUE: Invoice #INV-619682 - Immediate Action Required
```

正文声称发票逾期 86 天，要求 48 小时内查看附件并付款。真正的分析对象是 base64 编码的 SVG 附件 `Invoice_INV-619682.svg`。解码后能看到两层内容：一层是正常发票图形，用矩形、文本和金额构造可信外观；另一层是隐藏在 `<script><![CDATA[...]]></script>` 中的 JavaScript。

脚本的关键行为包括：

- 反自动化：检查 `navigator.webdriver`、`PhantomJS`、`HeadlessChrome`、`Selenium`、包含 `Burp` 的 UA 等，命中后跳到 `about:blank`。
- 环境指纹：用 Canvas、屏幕色深、语言、时区、硬件并发数等生成 `entropy` 和 `globalSeed`。
- 反调试：阻止 F12、Ctrl+U、Ctrl+Shift+I/J/C/K、右键菜单，并用 `debugger` 与 `performance.now()` 检测调试停顿。
- 诱饵数据：脚本中直接出现 `wmctf{fake_flag}`、`WMCTF{not_the_real_flag}` 等假 flag，不应作为结果。
- 混淆数据：`polymorphicData` 的前几项是诱饵，后几项通过 `charMap` 把形如 `4p2V`、`77iL`、`4py9` 的片段还原成字符，再与固定 key `WMCTF_2025_SVG_ANALYSIS` 做轻量 XOR。
- 干扰分支：脚本后段引用了 `segments`、`verification()`、`constructPayload()` 和 `window.hiddenData`，但这些符号在附件 SVG 中没有定义，不能把该分支当成有效执行路径；有效分析应围绕已定义的数组、映射表和诱饵数据做静态还原。

`mail_phishing/docker` 目录还包含一个 Ubuntu 16.04 + xinetd 的服务骨架，`ctf.xinetd` 把 `pwn` 通过 chroot 暴露在 `8888` 端口，`run.sh` 只执行 `./pwn`。该目录中的 `flag` 是占位值 `flag{AAAAAAAA_BBBBBBBBBBB_CCCCCCCCCCC}`，邮件分析的主要附件仍是 `.eml` 中的 SVG。

Cloudflare 报告里总结的 SVG 钓鱼模式和本题完全对应：SVG 可以作为邮件附件或云盘链接投递，外观像普通发票、语音留言或文档；打开后可以自动重定向到凭据收集站，也可以在 SVG 内直接渲染伪造登录页；脚本还常配合 Cloudflare Workers、Turnstile、一次性域名、Base64 参数、DOM 注入和反分析逻辑来绕过静态检测。本题把这些实战特征压缩成一个可审计的 `.svg` 样本，解法就是把 SVG 当作可执行文档做 JavaScript 去混淆，而不是把它当普通图片看。

## 方法总结

- `Shopping company` 的核心技巧是从“AI 客服能分析文件”定位到工具调用边界，再找工具函数里的危险动作。看到 AI agent 自动解压、自动分析、自动运行外部文件时，应优先检查 tool 代码，而不是只和模型提示词周旋。
- 多阶段内网题要把入口 RCE、内网探测、凭据复用、Open WebUI N-day、Redis/Gob 反序列化分开验证；每段都要有独立的命令执行或文件写入证据，避免把一段成功误当成全链路成功。
- `phishing email` 的识别信号是 `.eml` + SVG 附件 + 发票/催收社工文案。SVG 可以执行 JavaScript，排查时要解码 MIME 附件、查看 `<script>`、识别诱饵 flag 和反调试逻辑，再沿混淆数组、映射表、XOR key 等数据流还原真实信息。
