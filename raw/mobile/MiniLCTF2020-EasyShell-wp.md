# MiniLCTF2020 - EasyShell

## 题目简述

原 APK 使用腾讯 StubShell/乐固类壳：入口 DEX 只有壳代码，真实 DEX 放在 native 库与 assets 管理的载荷中。官方仓库没有题解，但同时保留了 `easyshell.apk` 与脱壳后的 `unpacked.apk`，因此可以从真实 DEX 还原完整状态机和 flag。

## 解题过程

列出原 APK 内容可以看到典型壳特征：

```text
com.tencent.StubShell.TxAppEntry
lib/armeabi/libshell-super.2019.so
assets/libshellx-super.2019.so
assets/0OO00l111l1l
```

原包 `classes.dex` 只有约 72 KiB，而 `unpacked.apk` 的 DEX 约 850 KiB。实际复现可在受控模拟器中让壳加载后 dump DEX；本仓库提供的 `unpacked.apk` 则是可直接核对的官方脱壳结果。

反编译 `MainActivity` 后可见两个关键字段：

```java
U = "mi{f4Ke";
W = "minil{unP4Ck_tH15_e4sY_897a5c}";
```

在主输入框提交 `mi{f4Ke` 会弹出第二个输入框。比较前，方法 `y()` 依次替换：

```text
1 -> l
8 -> 2
9 -> 3
a -> 1
C -> c
```

对 `W` 应用这组替换：

```python
s = 'minil{unP4Ck_tH15_e4sY_897a5c}'
s = s.replace('1', 'l').replace('8', '2').replace('9', '3')
s = s.replace('a', '1').replace('C', 'c')
print(s)
```

得到需要在弹窗中提交的真实 flag：

```text
minil{unP4ck_tHl5_e4sY_23715c}
```

按钮点击 25、40、200 次出现的提示只是引导，其中第 200 次明确显示 `unpack me.`。当前材料可以静态完成整个校验，不需要猜测无人解出时的远程状态。

## 方法总结

加壳 APK 要先区分壳 DEX 与运行时真实 DEX，不能在 StubShell 上浪费全部分析时间。脱壳后还要追踪 UI 状态机：硬编码字符串未必就是提交值，必须按真正比较前的替换顺序执行。
