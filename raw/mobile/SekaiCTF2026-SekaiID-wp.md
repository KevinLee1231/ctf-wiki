# SekaiID

## 题目简述

附件包含两个 Android APK：

- `SekaiID.apk`：数字会议证件钱包；
- `Verifier.apk`：工作人员扫描与管理端。

预期目标是让 Verifier 接受管理员身份并在 Admin Dashboard 显示 `/system/flag.txt`。源码还设计了隐式 Intent、导出 Provider、URI grant 和 claims tag 等 IPC 攻击面。

仓库没有官方 solution；APK 反编译得到的关键 manifest/类名和实机步骤可与[固定提交版本的参赛题解](https://github.com/Abdelkad3r/SekaiCTF-2026/blob/ef87965f07e32162595457285da138f181492f54/misc/sekaiid/README.md)交叉核对。

## 解题过程

### 确认 flag 的 UID 边界

静态检查两个 manifest，发现应用发布包仍设置：

```xml
android:debuggable="true"
```

Verifier 的 `AdminDashboardActivity` 在管理员页面调用 `IssuerRecoveryProvider.read()`；后者直接执行 `File(...).readText().trim()`，默认路径为：

```text
/system/flag.txt
```

`setup.sh` 先把 Verifier 安装为 `/system/app/SekaiVerifier` 下不可被不同签名 APK 替换的系统应用，取得其 UID 后再对 flag 执行 `chown <verifier_uid>` 和 `chmod 600`。因此普通 ADB shell（UID 2000）确实不能读文件，但 Verifier 自己的应用 UID 可以。

### 使用 `run-as` 直接切换应用身份

题目实例把 ADB 套在 TLS 上。先在本机把远端 TLS 流量转成 ADB 可连接的普通 TCP：

```bash
python3 tls_proxy.py <instance-host> 1337
adb connect 127.0.0.1:5555
```

基础系统在部署后已经删除 `su` 并把全局 `ro.debuggable` 改为 0，所以 `adb root` 不可用；但 `run-as` 检查的是目标 APK 在 PackageManager 中的 `FLAG_DEBUGGABLE`。Verifier 自己仍以 `android:debuggable="true"` 发布，故可直接切换到它的 UID：

```bash
adb shell run-as com.sekai.verifier cat /system/flag.txt
```

即可得到：

```text
SEKAI{welcome-to-the-conference!}
```

这条路径不需要构造 exploit APK。

### 未使用的 IPC 攻击面

反编译代码能解释题目要求“提交 exploit APK”的原始设计，但不应把它误写成实际必需步骤：

- Verifier 用没有 `setPackage` 的隐式 `ACTION_PRESENT_CREDENTIAL` 请求 presentation，攻击者 Activity 可注册同一 action 参与解析；
- 导出的 `BadgeShareActivity` 会把指向未导出 `CompanionProvider` 的 content URI 作为结果返回并附加读 grant；
- 导出的 `CompanionInfoProvider` 暴露 credential ID、公钥和 pairing fingerprint；
- `IdentityProviderActivity` 又检查调用包是否为 `com.sekai.verifier`，所以单纯抢占隐式 Intent 仍不够，还需处理调用者身份与 `FLAG_ACTIVITY_FORWARD_RESULT`/任务链；
- presentation 同时包含 ECDSA proof、revealed claims 和 `claimsTag`，Verifier 最终从 revealed `accessProfile` 决定进入哪个 dashboard。

APK 中的配对派生实现把 32 字节 seed 与

```text
SHA256("pairing-fingerprint:v2" || 0x1f || credentialId || 0x1f || publicKey)
```

异或生成 fingerprint，另一端可用同一流还原 seed；把该值通过导出 Provider 暴露会削弱 claims HMAC 的密钥边界。不过仓库没有端到端 exploit APK，第三方材料也只还原到预期 gadget 设计，不能据此声称完整 IPC 链已经被官方验证。实际解法在第一步 `run-as` 就已结束。

## 方法总结

决定性漏洞是 Android 平台的调试身份切换，所以按主障碍归入 Mobile 比原仓库的 Misc 更准确。敏感文件即使是 `0600`，只要归属 debuggable 应用 UID，ADB `run-as` 就能以该 UID 读取；关闭全局 `adb root` 并不能抵消应用自身的 debug flag。发布构建必须关闭 debuggable，同时还应把隐式 Intent 改为显式目标、收紧 Provider/grant，并让签名覆盖所有参与授权决策的 claims。
