# DownUnderCTF 2020 - flag-getter

## 题目简述

附件是经过混淆的 Android APK。界面上的四个按钮分别向服务器发送 HTTPS 请求，四段响应合起来才是 flag；应用使用 OkHttp `CertificatePinner` 固定证书，使普通中间人代理无法解密流量。题目重点是定位并绕过 APK 中的证书固定逻辑，然后观察应用真实请求，而不是完整手工还原大段字符串混淆。

## 解题过程

从官方源码可以确认程序结构：

```java
CertificatePinner certificatePinner = new CertificatePinner.Builder()
    .add(domain, "sha256/9BAH1tna31gGCVx9PiXNwZ23wezi9YDGBSiUflTu8dM=")
    .add(domain, "sha256/YLh1dUR9y6Kja30RrAn7JKnbQG/uEtLMkBgFF2Fuihg=")
    .add(domain, "sha256/Vjs8r4z+80wjNcr1YKepWQboSIRi63WsWXhIMN+eWys=")
    .build();

this.client = new OkHttpClient.Builder()
    .certificatePinner(certificatePinner)
    .build();
```

按钮分别调用 `H.H0()` 到 `H.H3()` 生成请求路径。`H.java` 用大量八进制字符和逐字节算术隐藏路径；逐个还原虽然可行，但成本很高，抓取运行时请求更直接。

先用 Apktool 解包：

```bash
apktool d flag-getter.apk -o flag-getter
grep -R "Pinned certificates\|certificate" flag-getter/smali*
```

官方 APK 中，证书校验最终落在一个签名类似下面的 Smali 方法中：

```smali
.method public final a(Ljava/lang/String;Ljava/util/List;)V
```

在方法体开头加入：

```smali
return-void
```

这样仍保留请求逻辑，只让 pinning 校验提前返回。随后重建并使用自己的测试密钥签名：

```bash
apktool b flag-getter -o patched-flag-getter.apk
keytool -genkeypair -keystore test.jks -alias test \
  -keyalg RSA -keysize 2048 -validity 3650
jarsigner -keystore test.jks patched-flag-getter.apk test
adb install patched-flag-getter.apk
```

在受控模拟器中安装代理 CA，配置 mitmproxy 或同类 HTTPS 代理，然后依次点击四个按钮。每个响应提供一段 flag，将响应按按钮顺序拼接后得到：

```text
DUCTF{n0t_s0_s3cre7_4ft3r_4LL_!!11!}
```

历史服务现已不一定在线，但仓库中的官方源码和 `flag.txt` 能交叉确认该结果。实际分析未知 APK 时，应只在隔离模拟器中运行，且不要把测试 CA 安装到日常设备。

## 方法总结

面对重度混淆且答案来自网络响应的 APK，先判断是否必须完全去混淆。本题只需保留业务请求并旁路证书固定，因此定位 pinning、做最小 Smali 补丁、重签名和抓包是更短的验证链。补丁后还要同时满足系统信任代理 CA、应用走代理和四个请求按顺序触发这三个条件。
