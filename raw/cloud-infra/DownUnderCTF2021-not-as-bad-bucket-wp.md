# DownUnderCTF 2021 - Not as Bad Bucket

## 题目简述

题目仍然使用 Google Cloud Storage 托管静态网站，但匿名访问桶根路径会返回拒绝。页面声称“只有登录员工才能访问秘密文件”，实际 IAM 配置却把 `roles/storage.objectViewer` 授予特殊主体 `allAuthenticatedUsers`：这代表任何通过 Google 身份认证的用户，而不是某个企业员工组。

```hcl
data "google_iam_policy" "viewer" {
  binding {
    role    = "roles/storage.objectViewer"
    members = ["allAuthenticatedUsers"]
  }
}
```

## 解题过程

浏览器匿名请求无法列举对象，是因为它没有携带 Google 身份。先用自己控制的普通 Google 账号完成 `gcloud auth login`，再通过 `gsutil` 发起认证请求即可进入 `allAuthenticatedUsers` 的授权范围。

从站点 URL 取出桶名后列举根目录：

```bash
gsutil ls 'gs://<bucket>'
```

输出显示一个 `pics/` 前缀：

```text
gs://<bucket>/index.html
gs://<bucket>/pics/
```

继续列举该前缀：

```bash
gsutil ls 'gs://<bucket>/pics/'
```

可以看到未在网页中引用的 `flag.txt`：

```text
gs://<bucket>/pics/flag.txt
gs://<bucket>/pics/lisa.jpg
```

下载并读取文本对象：

```bash
gsutil cp 'gs://<bucket>/pics/flag.txt' -
```

得到：

```text
DUCTF{all_AUTHENTICATED_users_means_ALL_AUTHENTICATED_USERS_silly}
```

## 方法总结

本题考查云 IAM 特殊主体的语义。`allAuthenticatedUsers` 不是“组织内所有用户”，而是“所有成功认证的 Google 身份”；它通常只比完全匿名的 `allUsers` 多一道登录门槛。遇到匿名 HTTP 被拒绝的公开云存储站点时，应继续使用已认证的官方 CLI 测试对象列举和读取权限，并检查授权对象究竟是组织域、具体服务账号，还是过宽的特殊主体。
