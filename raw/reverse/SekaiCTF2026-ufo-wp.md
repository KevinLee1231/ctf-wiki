# ufo

## 题目简述

附件是经过名称混淆的 Android 应用 Universal File Opener，包名为 `com.krauq.universalfileopener`。flag 不以字符串形式存放在 APK 中，而是由五类查看行为逐步解锁主界面上的隐藏像素层。题目提示最终文本需要自行包在 `SEKAI{}` 中。

源码仓库只提供 APK，没有官方解题脚本；关键交互路径可由反编译结果与[这份固定版本的参赛题解](https://github.com/NarainKrishna/bi0s_pentest/blob/16e4f9c552971421a06e291272d08e6b059ca066/CTFs/Sekai%20CTF%20%282026%29/ufo/ufo.md)相互核对。

## 解题过程

### 追踪隐藏状态

用 JADX 检查 DEX。虽然类名已经混淆，但偏好键字符串仍然稳定：类 `b7.w` 初始化了整型 DataStore 键：

```text
ambient_trace
```

继续跟踪该键的读取和更新，在 `b7.a` 附近可以还原五个按位标志：

```text
0x01 -> PDF Signature
0x02 -> PDF Metadata
0x04 -> Office Text
0x08 -> Spreadsheet
0x10 -> Image Transform
```

主界面根据已置位的标志逐层绘制彩色像素。全部显示所需状态为：

$$
0x01\mathbin{|}0x02\mathbin{|}0x04\mathbin{|}0x08\mathbin{|}0x10=31.
$$

可以实际打开五类文件逐项触发，也可以直接修改
`/data/data/com.krauq.universalfileopener/files/datastore/ufo_prefs.preferences_pb`。更方便的做法是用 Frida 挂钩负责合并状态的 `r7.m0.g(int, int, y0.k)`，把传入的新状态固定为 31：

```javascript
Java.perform(function () {
    const M0 = Java.use("r7.m0");
    const merge = M0.g.overload("int", "int", "y0.k");
    merge.implementation =
        function (_state, other, cont) {
            return merge.call(this, 31, other, cont);
        };
});
```

这里修改的是显示状态，不是绕过付费订阅；付费逻辑与 flag 路径无关。

### 重建缺失功能图案的 QR

状态为 31 后截取主界面的完整像素层，去掉背景和非有效色块，再把彩色格映射为黑白模块。得到的网格是 $21\times21$，恰好对应 QR Version 1，但挑战图故意省略了三个 finder、separator、timing、format information 和固定 dark module，因此普通相机不会直接识别。

按 Version 1 的标准坐标补回功能模块，对数据区按从右下角开始的双列之字形顺序读取，并尝试 8 种 mask。有效参数为纠错等级 L、mask pattern 5；去掩码后可以得到合法的 byte-mode 数据流，内容为：

```text
clankedormiscgod
```

按题目要求包裹后得到：

```text
SEKAI{clankedormiscgod}
```

## 方法总结

这是一道 APK 状态逆向与 QR 重建题，决定性障碍不是 Android 权限或付费逻辑，因此归入 Reverse 比仓库原先的 Misc 更准确。面对名称混淆，应优先追踪仍可读的偏好键、方法签名和状态数据流；识别出 $21\times21$ 网格后，也不能因为缺少三个定位框就否定 QR，而应按版本补齐功能模块，再由格式信息或枚举确定纠错等级与 mask。
