# Hijacker

## 题目简述

题目给出一个 Android PIN 登录应用，并要求提交另一个 APK 作为 PoC。判题器会安装并启动 PoC，随后打开目标应用，按照固定但对选手未知的六位 PIN 点击数字键，最后截取屏幕。取得截图中泄露的 PIN 后，再把它提交给独立的 `pin2flag` 服务即可获得 flag。

目标 APK 内的 `GLOBAL.PIN` 是 `000000`，旁边还明确写着“这是假的 PIN”，因此直接反编译目标程序并不能得到判题器实际输入的密码。决定性问题是 Android 悬浮窗权限：PoC 可以在目标应用上方放置一个外观相同且能够接收触摸的窗口。

## 解题过程

首先检查两个 APK 的清单。目标包名为 `com.aimar.id.hijacker`，登录界面是导出的 `LoginActivity`；官方 PoC 则声明了 `android.permission.SYSTEM_ALERT_WINDOW`，并由启动 Activity 拉起 `OverlayService`。判题器会尝试授予 PoC 声明的 Android 权限，因此悬浮窗权限在测试环境中可直接使用。

目标应用的登录逻辑并不复杂：点击数字键后把数字追加到内部字符串，同时只在六个文本框中显示 `*`；输入满六位后与假的 `GLOBAL.PIN` 比较，不匹配就清空。真正有用的信息只存在于判题器产生的触摸序列中。

官方 PoC 的 `OverlayService` 复用了与目标登录页相同的 `activity_login` 布局，然后创建全屏悬浮窗：

```java
WindowManager.LayoutParams params = new WindowManager.LayoutParams(
    MATCH_PARENT,
    MATCH_PARENT,
    2038,                    // TYPE_APPLICATION_OVERLAY
    0x400,                   // FLAG_FULLSCREEN
    PixelFormat.TRANSLUCENT
);
windowManager.addView(overlayView, params);
```

窗口类型十进制 `2038` 即 `TYPE_APPLICATION_OVERLAY`。由于该窗口覆盖整个屏幕且没有设置 `FLAG_NOT_TOUCHABLE`，判题器对下层目标应用坐标的点击会先落到 PoC 自己的按钮上。视觉上仍然是一模一样的 PIN 键盘，所以判题器不会因为界面变化而改用其他坐标。

PoC 与目标程序最关键的差别是显示策略。每个数字按钮的监听器把按钮文字原样写进当前位置，而不是写入星号：

```java
int digit = Integer.parseInt(((Button) view).getText().toString());
pinTexts[pinIndex].setText(String.valueOf(digit));
pinIndex = (pinIndex + 1) % 6;
```

清除键则把六个文本框清空并令 `pinIndex = 0`。这一点不能省略，因为判题器会以约 $15\%$ 的概率故意点击一个错误数字，再点击 `C` 并从头输入，用来排除只记录最后六次触摸却不处理清除操作的粗糙方案。

判题流程最终输入的 PIN 是 `593720`。六次正确点击结束后，PoC 的悬浮页会把这六个数字直接留在屏幕上，判题器截取的图片因此完整显示密码。官方 APK 中虽然还保留了一个向临时 ngrok 地址发送数字的 `sendToServer()` 方法，但按钮回调没有调用它；真正可靠、也与判题器设计一致的泄露通道是最终截图。

最后连接 `pin2flag` 服务，完成其 proof of work 后提交：

```text
PIN: 593720
```

服务端比较成功后输出本题 flag。

## 方法总结

本题不是从 APK 常量中寻找静态密码，而是利用 Android 的应用上层窗口劫持用户输入。完整利用链为：申请悬浮窗权限，启动长期存在的 Service，使用 `TYPE_APPLICATION_OVERLAY` 放置与目标界面相同的可触摸全屏布局，正确处理数字键和清除键，再通过判题器截图读取真实 PIN。

分析这类移动题时应同时检查目标 APK、PoC 运行方式和判题器。只看目标程序会被假 PIN 误导；只看官方 PoC 中未被调用的网络函数，也会错误理解数据外传方式。这里决定成败的是 Android 窗口层级与触摸分发机制，因此归入 Mobile，而不是普通 Misc。
