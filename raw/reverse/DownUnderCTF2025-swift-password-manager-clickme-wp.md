# SwiftPasswordManager: ClickMe

## 题目简述

题目是 SwiftCrossUI 编写的桌面密码管理器。界面中存在 `Flag` 按钮，点击处理器本身会用逐字节异或还原 flag 并弹出 alert，但视图构造末尾追加了无参数的 `.disabled()`，所以正常 UI 无法点击。

关键不在密码文件功能，也不需要从数组手工抄出 flag：需要逆向 Swift 符号找到禁用视图的调用，并在运行时把“禁用”为真的参数改为假。

## 解题过程

### 确认按钮逻辑可用但被禁用

`ContentView` 中的按钮处理器计算：

```swift
for i in 0..<a.count {
    let v = UInt8((38 * i + 17) & 0xff)
    o.append(v ^ a[i])
}
presentAlert(String(bytes: o, encoding: .utf8)!)
```

这表明只要处理器能被调用就会显示结果。问题紧随其后：

```swift
Button("Flag") { /* decode and present alert */ }
    .disabled()
```

官方题解使用 `nm` 在 Swift 符号中定位 `SwiftCrossUI.View.disabled` 的特化函数。无参数版本把 button 的禁用布尔值传入该函数，故在 x86-64 调用约定下令首个整数参数寄存器 `rdi` 为 0，就能让 `.disabled()` 表现为未禁用。

### 在调试器中修改参数

官方给出的 GDB hook 如下：

```gdb
break $s12SwiftCrossUI4ViewPAAE8disabledyQrSbF
commands
    set $rdi = 0
    continue
end
run
```

命中 `disabled` 后继续运行，`Flag` 按钮即可点击；点击后使用原程序的 XOR 循环而非外部脚本生成弹窗中的 flag。若采用二进制 patch，目标同样是使传给该函数的 `Bool` 恒为 false，而非修改 alert 或伪造输出。

### 验证

源码的唯一成功路径是 `Flag` 点击回调中的异或解码和 `presentAlert`；官方 WRITEUP 的 hook 正好只改变该回调到达前的 UI 状态。本文未启动 Swift 应用或调试器，验证依据为官方 hook 与源代码的对应关系。

## 方法总结

- 核心技巧：区分“按钮的业务回调”与“视图层可达性”；本题只需解除后者。
- 识别信号：Swift/SwiftUI 程序有灰色按钮、而符号中能找到 `disabled` 或 `isEnabled` 时，优先跟踪该布尔值和调用约定。
- 复用要点：最小 hook/patch 应维持原始回调和解码过程，以弹窗的原生输出作为验证，而不是从 UI 外另行复刻结果。
