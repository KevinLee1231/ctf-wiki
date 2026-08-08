# EzJvav

## 题目简述

题目是一个主题发布站：创建 workspace 后上传 ZIP，访问 `/site/<id>/preview` 预览，访问 `/site/<id>/export` 导出。ZIP 路径检查本身不允许 ZIP Slip，却特意允许两个额外源文件：`preview/ThemeHook.java` 与 `preview/ReviewAdapter.java`。前者在隔离 worker 中编译执行，只能返回 `PreviewPatch` 修改 `headline`、`note`、`accent`；后者经严格源码过滤和编译后成为 Tomcat Valve 的受控发布组件。

关键不是直接 RCE。预览服务在调用 ThemeHook 前会临时把内部 `previewHandle` 和 `witness` 加在 CSS URL 的 `_h`、`_k` 参数中。恶意 ThemeHook 可以读出 `model.getAssetCss()` 并把它们搬到可见字段，从而满足后续 helper 产生 review token 的条件；之后按服务端状态机完成发布，第二次导出的响应头给出 receipt/flag。决定性障碍是公开 Web 工作流中的状态泄露与多阶段验证，故归 `web`。

## 解题过程

### 最小主题与状态泄露

ZIP 至少包含 `manifest.json`、模板和 CSS，以及两个允许的 Java 文件。创建 workspace 后将 ZIP 作为 multipart 字段 `archive` 提交到返回的 `uploadUrl`。`ThemeHook` 实现题目接口并返回 `PreviewPatch`；其目标不是访问系统，而是把 `_h`、`_k` 原样反映到渲染字段，并放入一个自选的 16--64 位十六进制 `clientKey`。

服务端关键顺序为：

```java
model.setAssetCss("/site/" + siteId + "/asset?path=assets/theme.css");
String handle = issuePreviewHandle(siteRoot);
String witness = previewWitness(siteRoot, handle);
model.setAssetCss(original + "?_h=" + handle + "&_k=" + witness);
runPresentationPass(siteRoot, model, servletContext); // ThemeHook
restoreAssetCss(model, stamp);
reviewToken = acceptPreviewSignal(handle, note, headline, accent);
```

于是 ThemeHook 要解析 asset URL 中的 `handle` 与 `witness`，并选择两个长度为 16--64 的十六进制字符串 `stageToken`、`clientKey`。随后按下一节的 helper 格式写入 `note`、`headline`、`accent`。预览响应 HTML 经过转义后仍能读回这些字段；`reviewToken` 不会作为独立响应头返回，而是 ThemeHook 输入通过 helper 校验后被服务端绑定到该主题，供后续 Valve 请求使用，因此 exp 应按同一公式在本地计算它。

### 计算辅助校验并完成发布

`studio-agent.bin` 对应的 Rust helper 和本题 `.build-config.env` 共同规定 preview 校验。不能直接采用 `build.rs` 中的默认值；当前构建实际使用以下切片参数：

```text
tail_handle=4       tail_witness=2      head_token=4
head_client=3       head_witness=4      head_handle=3
hash_head=3         hash_tail=3         preview_slice_start=1
install_slice_start=5
```

这里 `stageToken` 是 ThemeHook 自选的输入，`reviewToken` 是 helper 校验成功后返回给服务端并用于安装 Valve 的输出。准确条件为：

```text
note     = stageToken + ":" + clientKey
headline 以 tail(handle,4) + tail(witness,2) + head(stageToken,4) 结尾
accent   = "#" + head(clientKey,3) + head(witness,4) + head(handle,3)

visibleWitness = tail(witness,2) + head(witness,4)
digest = SHA-256("review:" + head(handle,3) + ":" + tail(handle,3)
                  + ":" + visibleWitness + ":" + clientKey + ":" + stageToken)
reviewToken = digest[1:25]
```

上式切片采用 Python 的左闭右开记法，所以 `reviewToken` 为 24 个十六进制字符。ThemeHook 首次提交的是借 `_h/_k` 触发 helper 的字段组合；服务端会把真实 `witness` 作为 helper 第五个参数，攻击者不能自选它，但必须从预览 URL 泄露并纳入 `headline`、`accent` 和本地 token 计算。

`ReviewAdapter.java` 的正确作用是让服务端接受并安装受限 Valve；它不需要自行偷取 header 或执行命令。安装成功后，必须严格按时间窗完成：

1. **第一次** `GET /site/<id>/export`：保存 `X-Site-Index`。服务端以它和预览期的 render/index label 计算 `bundleDigest`。
2. `GET /site/<id>/preview`，带 `X-Theme-Mode: peek`、`X-Theme-Key: <reviewToken>`：取得 `theme_flow=<challenge>` cookie、`X-Theme-Tag: <marker>`、`X-Theme-Flow: <installWitness>`。
3. 按 helper 安装校验计算 `proof`：

```text
installDigest = SHA-256(clientKey + ":" + reviewToken + ":" + reverse(challenge)
                         + ":" + marker + ":" + bundleDigest
                         + ":" + installWitness)
proof = installDigest[5:37]
```

4. 在 5 秒 challenge 窗口且距离第一次 export 不超过 12 秒内再次访问 preview，带 `X-Theme-Mode: apply`、`X-Theme-Key`、`X-Theme-Tag`、`X-Theme-Match: <proof>` 和 `theme_flow` cookie。
5. apply 成功会设置 HttpOnly `theme_publish` cookie；在其 5 秒内第二次 `GET /site/<id>/export`，从 `X-Publication-Receipt` 取得最终值。

该顺序源自 `sealBundle`、`openPass`、`acceptPass` 和 `consumePendingReceipt`：跳过第一次 export 时 `bundleDigest` 不存在，apply 必然失败。

### 验证

检查点依次是：预览 HTML 中确实出现 `_h/_k` 派生字段；第一次 export 有 `X-Site-Index`；peek 返回三个状态材料；apply 返回 `theme_publish`；第二次 export 出现 `X-Publication-Receipt`。原题解没有保存可运行 exp、具体 helper 二进制逆向输出或比赛回包，因此本文不伪造最终 receipt/flag；流程和公式均以题目 Java 源码与随附 helper 模板为依据。

## 方法总结

- 核心技巧：将受限 ThemeHook 当作内部状态泄露器，而非直接 RCE，并按服务端绑定的短时发布状态机完成 proof。
- 识别信号：上传系统“特例允许”可编译 hook、预览模型中出现临时 token、导出接口携带中间 digest，以及多组 cookie/header 必须短时匹配。
- 复用要点：审计状态机要画出每个值的产生、绑定、TTL 与消费位置；不要被命名类似 flag/catalog 的 decoy 状态或 ZIP Slip 表象带偏。
