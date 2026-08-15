# Flagapp

## 题目简述

附件是一个 Flutter Android 应用。APK 的包名为 `com.example.flagapp`，启动 Activity 为 `com.example.flagapp.MainActivity`，同时提供 ARM64、ARMv7 和 x86_64 原生库。应用界面有一个 `Generate Flag` 按钮，但 flag 不会以完整明文静态保存在 APK 的普通字符串表中。

官方预期是动态分析：运行应用并点击按钮，让 Dart 代码在运行时构造 flag，再转储或扫描进程内存中的 `shellmates{...}` 字符串。

## 解题过程

先确认包信息并安装 APK：

```bash
aapt dump badging flagapp.apk | grep -E '^(package|launchable-activity|native-code)'
adb install -r flagapp.apk
adb shell am start -n com.example.flagapp/.MainActivity
```

输出中的关键信息为：

```text
package: name='com.example.flagapp'
launchable-activity: name='com.example.flagapp.MainActivity'
native-code: 'arm64-v8a' 'armeabi-v7a' 'x86_64'
```

在模拟器或真机中点击 `Generate Flag`。这一步不能省略，因为它会触发 Dart 逻辑创建目标字符串；只在应用启动前搜索内存可能找不到结果。

可以在已运行 Frida Server 的测试设备上，用下面的 `scan.js` 扫描进程可读写内存。搜索模式是 `shellmates{` 的 ASCII 字节：

```javascript
const pattern = "73 68 65 6c 6c 6d 61 74 65 73 7b";

for (const range of Process.enumerateRanges({
  protection: "rw-",
  coalesce: true,
})) {
  Memory.scan(range.base, range.size, pattern, {
    onMatch(address) {
      console.log(`match at ${address}`);
      console.log(hexdump(address, {
        length: 96,
        header: false,
        ansi: false,
      }));
    },
    onError(reason) {
      console.error(`${range.base}: ${reason}`);
    },
    onComplete() {},
  });
}
```

点击按钮后附加到进程：

```bash
frida -U -n com.example.flagapp -l scan.js
```

若使用完整内存转储工具，流程同样是先点击按钮，再 dump `com.example.flagapp`，最后在所有转储片段中搜索：

```bash
rg -a -o 'shellmates\{[^}]{1,128}\}' dump
```

内存中可恢复出：

```text
shellmates{dYn4M1c_4N4lys1S_1n_FLuTt3r_1s_d0P3}
```

界面截图只会显示普通按钮或可转写文本，不提供额外的空间、图形或载体证据，因此不需要作为题解图片保留。

## 方法总结

Flutter 发布构建通常把 Dart AOT 代码放入 `libapp.so`，静态分析仍然可行，但对象布局、运行时字符串和符号恢复成本可能高于直接观察进程。本题按钮会在运行时生成完整 flag，所以以已知前缀扫描堆内存是更短、更符合题意的路径。

动态分析时要控制采样时机：目标对象必须已经创建且尚未被垃圾回收。若一次扫描没有结果，应保持生成结果所在页面、重新点击按钮并立即扫描，而不是直接断言 APK 中不存在 flag。上述操作应只在自有模拟器或获授权测试设备上进行。
