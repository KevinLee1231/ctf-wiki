# co2v2

## 题目简述

这一版移除了前题直接读 flag 的接口，却保留了 `/save_feedback` 的 Python 类污染。攻击链要同时解决三件事：把 CSP nonce 变为可预测值、重建一个关闭自动转义的 Jinja2 环境，以及让携带管理员会话的报告机器人访问公开文章。最终是受 CSP 约束的存储型 XSS 窃取非 `HttpOnly` 会话 Cookie。

## 解题过程

先注册并登录普通用户。向 `/save_feedback` 提交的类污染载荷如下：

```json
{
  "title": "",
  "content": "",
  "rating": "",
  "referred": "",
  "__class__": {
    "__init__": {
      "__globals__": {
        "RANDOM_COUNT": 0,
        "SECRET_NONCE": "t",
        "TEMPLATES_ESCAPE_ALL": false
      }
    }
  }
}
```

`generate_nonce(path)` 原本计算 `SHA256(SECRET_NONCE + path + random_suffix)`。将 `RANDOM_COUNT` 置零后随机后缀为空；首页的 nonce 因而固定为 `SHA256("t/")`，即：

```text
a2fe8952412bc49de813bb82db50d5aa497d6106b6b43c8a72cab443aa017e32
```

仅修改全局变量还不够：`template_env.env` 已在启动时创建。以普通登录用户向 `/admin/update-accepted-templates` 提交 `{"policy":"strict"}`，接口会重新创建环境，却错误地直接读取已经被污染的 `TEMPLATES_ESCAPE_ALL`；结果是新环境关闭了自动转义。

接着创建一篇公开文章，把题内已知 nonce 的 `<script>` 放进标题或正文。脚本以同源 `fetch` 读取需要管理员会话的内容，并仅在授权的自有收集端记录结果；不在题解中固化临时收集地址。调用 `/api/v1/report` 后，服务端让管理员机器人访问首页。文章未被转义、脚本 nonce 又与 CSP 匹配，脚本会在管理员浏览器中运行；会话 Cookie 未设置 `HttpOnly`，故可取得管理员会话并访问 flag。

最终 flag 为：

```text
DUCTF{_1_d3cid3_wh4ts_esc4p3d_}
```

## 方法总结

安全机制必须作为整体审计：CSP nonce 只有在秘密不可被应用数据改写时才有意义，自动转义只有在每个实际使用的模板环境中开启才有效，Cookie 的 `HttpOnly` 也不能替代 XSS 修复。根因仍是任意递归属性合并；对输入做字段白名单、拒绝魔术属性，并将管理员专用配置接口做真正的授权检查，才能切断整条链。
