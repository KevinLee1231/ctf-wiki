# Flamecamp

## 题目简述

题目只给出 Android 安装包。应用界面是一页 Firebase 邮箱登录表单，没有注册入口；登录成功后，它会从 Firebase Storage 下载并显示 `flamecamp_welcome.jpg`。因此 APK 只是访问云资源的客户端，真正的限制是“必须拥有可通过 Firebase Authentication 验证的账号”。

反编译后可从 `res/values/strings.xml` 读到 Firebase 客户端配置：

```text
project_id = flamecamp
google_storage_bucket = flamecamp.appspot.com
google_app_id = 1:1090871602799:android:c1d198201cd7635db5279a
google_api_key = AIzaSyANqtzq5t7AJ1FDJjK9if77WsV_ebvktiA
```

Firebase 的 Web API Key 本身不是服务端秘密，它用于标识项目；漏洞在于项目允许任意用户通过公开的 Identity Toolkit 接口创建账号，而 Storage 规则又把“已认证”错误地等同于“有权读取秘密文件”。

## 解题过程

### 还原身份到资源的访问链

`MainActivity` 调用 `FirebaseAuth.signInWithEmailAndPassword()`。认证成功后，程序获取默认 Storage 实例，并访问根目录下的：

```text
flamecamp_welcome.jpg
```

最小权限链为：

```text
任意新建 Firebase 用户
  -> 获得有效 ID Token
  -> 满足 Storage 的 authenticated 条件
  -> 读取 flamecamp_welcome.jpg
```

应用没有注册按钮并不代表后端关闭了注册。使用 APK 中公开的 API Key 调用 Firebase Authentication REST API，即可创建一个题目项目内的邮箱用户：

```bash
curl 'https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=AIzaSyANqtzq5t7AJ1FDJjK9if77WsV_ebvktiA' \
  -H 'Content-Type: application/json' \
  --data '{
    "email": "player@example.com",
    "password": "Flamecamp-2023!",
    "returnSecureToken": true
  }'
```

成功响应会包含 `idToken`、`localId` 和邮箱等字段。也可以直接把这组邮箱与密码填回 APK 的登录页面，让应用完成后续下载；若用 REST 客户端复现，则将返回的 ID Token 作为 Bearer 凭据请求对应 Storage 对象。

登录成功后，欢迎图片中给出：

```text
UMDCTF{W3lc0m3_t0_Th3_$ecr3t_f1reb@se}
```

### 为什么 API Key 不是根因

移动应用中的 Firebase API Key 必须随客户端发布，因此提取到它只完成了项目定位。真正的组合缺陷有两点：

1. Authentication 仍允许公开创建邮箱账号；
2. Storage 规则仅检查 `request.auth != null`，没有进一步限定用户、角色或对象。

如果只泄露 API Key，但后端关闭自助注册并为对象设置了细粒度授权，攻击链就无法成立。

## 方法总结

- 核心技巧：不要停留在 APK 登录界面，而要还原 Firebase 的 `identity -> token -> storage object` 授权链。
- 识别信号：客户端没有注册入口，但包含完整 Firebase 项目配置且只要求“已登录”即可读取敏感对象时，应直接核对公开注册端点和 Storage 规则。
- 防御要点：API Key 不能替代授权；应关闭不需要的登录方式，并按用户、角色或自定义 claim 对具体对象实施最小权限控制。
