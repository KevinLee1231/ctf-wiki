# Chungus Vault 2.0

## 题目简述

页面已经填好管理员用户名与密码，但点击登录仍失败。混淆 JavaScript 实际没有读取这两个输入框，而是比较隐藏字段 `#check` 的 SHA-256 值；默认值代表普通用户身份。

## 解题过程

HTML 中隐藏字段为：

```text
e9b7a334826ff2ff28b066a92ba8734cb1134720fc5879592247f47b8d2a17d7
```

它等于 `SHA256("user-login")`。管理员分支比较的值是：

```text
dd668f583f3cc97929dc8bba5090c143b579314bfb7d32888febd09015e65b85
```

即 `SHA256("admin-login")`。在开发者工具中替换隐藏字段并点击按钮：

```javascript
document.querySelector("#check").value =
  "dd668f583f3cc97929dc8bba5090c143b579314bfb7d32888febd09015e65b85";
document.querySelector("#submitButton").click();
```

混淆脚本进入管理员分支后拼出：

```text
UMDCTF-{AdM1n_Byp4ss_Suc3ssfu1}
```

该字符串的 SHA-256 与 README 记录完全一致。

## 方法总结

把角色判断放在可修改的 DOM 隐藏字段中等于信任客户端。分析混淆代码时不必先还原所有无关字符串，只需沿按钮事件、条件比较和成功消息做数据流切片。
