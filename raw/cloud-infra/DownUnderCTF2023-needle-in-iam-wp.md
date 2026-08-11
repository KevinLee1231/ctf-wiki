# DownUnderCTF 2023 Needle In IAM Writeup

## 题目简述

题目给出一个 GCP 服务账号密钥，并说明 flag 位于自定义 IAM 角色 `ComputeOperator` 的描述中。直接执行 `gcloud iam roles describe` 会因权限不足失败，但账号拥有 `iam.roles.list` 权限。

## 解题过程

先使用题目提供的密钥激活服务账号：

```bash
gcloud auth activate-service-account --key-file=credentials.json
```

Terraform 源码说明了权限差异：服务账号绑定的自定义角色只包含 `iam.roles.list`；目标角色的 `description` 则直接被设置为 flag。虽然 `describe` 无权访问单个角色的详细接口，`list` 的每条结果同样包含描述字段。

用题目实例对应的项目 ID 执行列表查询，并按描述前缀过滤：

```bash
gcloud iam roles list --project=<PROJECT> --filter="description:DUCTF"
```

返回的 `ComputeOperator` 条目中可以读到：

```text
DUCTF{D3scr1be_L1ST_Wh4ts_th3_d1fference_FDyIMbnDmX}
```

## 方法总结

云权限审计不能只看资源名称，还要比较不同 API 返回的字段集合。本题不是云平台漏洞，而是最小权限配置没有考虑 `list` 与 `describe` 的信息重叠：禁止详细查询并不代表列表接口不会泄露同一描述字段。
