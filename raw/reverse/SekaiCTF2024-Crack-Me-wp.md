# Crack Me

## 题目简述

附件是一个 React Native Android 应用。界面要求输入邮箱和密码，并提供一个“Admin Account”复选框，但该复选框只是干扰项。真正目标是从 APK 内的 JavaScript bundle 恢复管理员凭据，登录 Firebase Authentication，再从当前用户对应的 Realtime Database 路径读取 flag。

虽然载体是 APK，决定性工作不是 Android 组件或权限利用，而是还原 React Native 业务逻辑及 AES 校验，因此归入 Reverse。

## 解题过程

### 1. 提取 React Native bundle

把 APK 当作 ZIP 解压后，可以在下面的位置找到主 JavaScript bundle：

```text
assets/index.android.bundle
```

这类 bundle 不是普通 Java/Kotlin 字节码，直接用 JADX 看不到核心登录逻辑。可以使用 [`react-native-decompiler`](https://github.com/richardfuca/react-native-decompiler) 按模块拆分：

```bash
npm start -- -i ./index.android.bundle -o ./output
```

输出共有数百个模块，大部分来自框架。搜索界面上的固定字符串 `Admin Account`，即可定位到保存主要页面逻辑的 `443.js`。

### 2. 恢复管理员账号和密码

登录按钮最终调用 `_verifyEmail()`。去掉异步和界面状态管理后，管理员判断可化简为：

```js
if (email !== 'admin@sekai.team' || !validatePassword(password)) {
    console.log('Not an admin account.');
} else {
    console.log('You are an admin...This could be useful.');
}
```

因此管理员邮箱固定为：

```text
admin@sekai.team
```

`validatePassword()` 要求密码长度为 17，并用 CryptoJS AES 加密后比较十六进制密文：

```js
AES.encrypt(password, Utf8.parse(KEY), {
    iv: Utf8.parse(IV)
}).ciphertext.toString(Hex) ===
'03afaa672ff078c63d5bdb0ea08be12b09ea53ea822cd2acef36da5b279b9524'
```

常量模块给出：

```text
KEY = react_native_expo_version_47.0.0
IV  = __sekaictf2023__
```

CryptoJS 在这里使用 AES-CBC 与 PKCS#7 padding。直接解密即可恢复密码：

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = b"react_native_expo_version_47.0.0"
iv = b"__sekaictf2023__"
ct = bytes.fromhex(
    "03afaa672ff078c63d5bdb0ea08be12b"
    "09ea53ea822cd2acef36da5b279b9524"
)

password = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16)
print(password.decode())
```

输出为：

```text
s3cr3t_SEKAI_P@ss
```

### 3. 让回调真正显示数据库内容

程序内还包含 Firebase 项目配置。成功调用 `signInWithEmailAndPassword()` 后，它访问：

```js
ref(database, `users/${user.uid}/flag`)
```

原始监听器忽略了快照参数，只弹出 `Keep digging, you're almost there!`：

```js
onValue(flagRef, () => {
    setState({
        errorTitle: 'Hello Admin',
        errorMessage: "Keep digging, you're almost there!"
    });
});
```

Firebase Realtime Database 的 value 事件会把当前位置的数据封装在快照中，调用 `snapshot.val()` 即可取出实际值。因此把 bundle 改成：

```js
onValue(flagRef, (snapshot) => {
    setState({
        errorTitle: 'Hello Admin',
        errorMessage: snapshot.val()
    });
    AlertPro.open();
});
```

该修改既可以把 flag 放进界面，也可以改为 `console.log(snapshot.val())` 后通过 `adb logcat` 读取。无需依赖 Firebase 文档的外部示例，关键语义就是“监听回调收到 DataSnapshot，`val()` 返回该节点内容”。

### 4. 重打包、签名并登录

替换 `assets/index.android.bundle` 后重新压缩 APK。修改破坏了原签名，必须重新对齐和签名；可使用 [`uber-apk-signer`](https://github.com/patrickfav/uber-apk-signer) 一次完成 v2 签名与对齐。只用传统 `jarsigner` 生成 v1 签名，在目标 API 配置下并不可靠。

若安装时报错：

```text
Targeting R+ requires resources.arsc to be stored uncompressed
and aligned on a 4-byte boundary
```

应正确执行 zipalign，或使用 API 30 以下的模拟器运行官方解法提供的重签名 APK。启动后输入管理员邮箱和恢复出的密码，修改后的数据库回调会显示：

```text
SEKAI{15_React_N@71v3_R3v3rs3_H@RD???}
```

## 方法总结

本题的关键路线是：识别 React Native 应用，提取 `index.android.bundle`，用界面字符串快速定位业务模块，从静态 AES 校验中恢复管理员密码，再修补 Firebase `onValue` 回调以读取 `snapshot.val()`。管理员复选框和 APK 中的 Java 外壳都不是核心。

外部工具只负责 bundle 拆分与 APK 重签名；管理员邮箱、密码算法、Firebase 路径以及快照取值方式都已经在正文中给出。即使不阅读外部链接，也可以完整复现分析与修改过程。
