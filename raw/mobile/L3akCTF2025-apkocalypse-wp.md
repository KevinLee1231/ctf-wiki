# L3akCTF 2025 Apkocalypse Writeup

## 题目简述

Apkocalypse 是一款看似普通的 Android 文件存储应用。Java 层会阻止应用在 root 环境运行；点击 “store flag” 后，`libstoreftw.so` 又会启动检测线程。native 主逻辑在内存中还原 flag，将其分两次写入应用私有目录的 `flag.txt`，随后立即调用 `unlink()` 删除文件。

因此，解题目标不是在删除后恢复文件，而是绕过两层检测，并在数据到达 `fwrite()` 时截获明文。官方解题材料中的应用界面和反编译器截图只展示了文字、按钮与代码，没有不可替代的视觉信息，本文将相关内容直接转写为文本。

## 解题过程

### 分析 Java 入口

反编译 `filestorage.apk` 后，可以看到包名为：

```text
ctf.l3akctf.filestorage
```

`MainActivity` 加载 native 库：

```java
static {
    System.loadLibrary("storeftw");
}
```

`onCreate()` 先调用：

```java
RootDetection.isDeviceRooted(this)
```

若返回 `true`，Activity 会显示提示并立即结束。另一个按钮则调用 native 方法 `storeftw()`。用 Frida 把 root 检测固定为 `false`：

```javascript
Java.performNow(function () {
    const RootDetection =
        Java.use("ctf.l3akctf.filestorage.RootDetection");

    RootDetection.isDeviceRooted.implementation = function (context) {
        return false;
    };
});
```

这里只绕过了 Java 层。直接点击按钮仍可能崩溃，因为 native 库还创建了独立检测线程。

### 找到写入后立即删除的窗口

分析 `libstoreftw.so` 的 `storeftw()` 可以看到，大段算术混淆最终生成一段明文，然后执行：

```c
FILE *fp = fopen(flag_path, "w");
if (fp != NULL) {
    fwrite("L3AK{", 1, 5, fp);
    fwrite(decrypted + 5, 1, decrypted_len - 5, fp);
    fclose(fp);
}
unlink(flag_path);
```

这个顺序揭示了两个关键事实：

1. flag 已经在传给 `fwrite()` 前以明文存在于内存中；
2. `unlink()` 只删除目录项，不会抹掉此前传给函数的参数。

与其完整逆向大段混淆算术，不如在明文跨越 libc API 边界时截获它。

### 延迟 native 检测线程

题目提供版本中，检测线程入口相对 `libstoreftw.so` 基址的偏移为 `0x627d0`。在该线程入口暂停 5 秒，可以让执行 flag 写入的主路径先完成：

```javascript
setTimeout(function () {
    const module = Process.findModuleByName("libstoreftw.so");
    if (module === null) {
        return;
    }

    const detector = module.base.add(0x627d0);
    Interceptor.attach(detector, {
        onEnter: function () {
            Thread.sleep(5);
        }
    });
}, 200);
```

`0x627d0` 是本题所附 APK、对应 ABI 下的相对偏移，不应视为其他编译版本也恒定的地址。实际分析时应依据所选 `.so` 的函数边界重新确认。

### Hook fwrite 重组两段 flag

`fwrite` 的原型为：

```c
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
```

本次写入的字节数是 `size * nmemb`，数据地址则是第一个参数。下面的 hook 按 native 代码的两次连续写入重组结果：

```javascript
let waitingForTail = false;

const fwrite = Module.findExportByName("libc.so", "fwrite");
Interceptor.attach(fwrite, {
    onEnter: function (args) {
        const length = args[1].toInt32() * args[2].toInt32();
        const text = Memory.readUtf8String(args[0], length);

        if (text === "L3AK{") {
            waitingForTail = true;
            return;
        }

        if (waitingForTail) {
            console.log("L3AK{" + text);
            waitingForTail = false;
        }
    }
});
```

把 Java root 绕过、检测线程延迟和 `fwrite` hook 合并为一个 Frida 脚本，然后启动应用：

```bash
frida -U -f ctf.l3akctf.filestorage -l solve.js
```

恢复进程并点击 “store flag” 后，控制台输出：

```text
L3AK{2fca62dde10486253541959b40635826}
```

该结果也与 native 代码中第一段固定前缀、第二段剩余内容的写入方式一致。

## 方法总结

本题的核心是选择合适的观察边界。算术混淆负责在内存中生成数据，但程序最终必须把明文交给 `fwrite()`；在这个稳定 API 上 hook，比逐条还原混淆指令更短、更可靠。随后的 `unlink()` 不会影响已经截获的参数。

Android native 题常同时存在 Java 环境检查和 C/C++ 反调试线程。前者可以在 ART 层改返回值，后者则需要在模块加载后按相对偏移定位。编写 hook 时还要严格遵循真实函数签名，并注意一次逻辑数据可能被拆成多次写入；本题若只筛选单次调用，就会分别看到 `L3AK{` 与剩余字符串，而不是完整 flag。
