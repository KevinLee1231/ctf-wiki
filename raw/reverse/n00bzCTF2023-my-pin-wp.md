# My-pin

## 题目简述

题目只提供 Java Swing JAR，没有官方 WP。界面只有 `0`、`1` 和重置按钮；真正逻辑位于 `Secret` 类中，九次按键会更新一组位数组并将其解码成界面文本。

## 解题过程

列出 JAR 后可见 `Mypin.class`、`PinButton.class`、`ResetButton.class` 和 `Secret.class`。反编译 `Secret.process(char)`，可以看到每个输入位与 `mydata` 当前列、上一行进位相加，再写回模 2 结果：

```text
value = carry + mydata[index] + (button - '0')
mydata[index] = value % 2
carry = 1 if value >= 2 else 0
```

`cnt` 从 1 增到 9，输入空间只有 $2^9=512$。按字节码完整模拟 `process` 和 `getData`，枚举九位串并筛选含 `n00bz{` 的输出：

```python
for pin in product("01", repeat=9):
    state = fresh_secret_state()
    for bit in pin:
        state.process(bit)
    text = state.get_data()
    if "n00bz{" in text:
        print("".join(pin), text)
```

`010110100` 可以解出 flag；最后一位改为 `1` 在该实现中也得到同一有效文本。完整结果为：

```text
n00bz{y0uuu_n33d_t0_bRutefoRc3_1s_e4zyY_}
```

## 方法总结

缺少题解时，GUI 只是入口，核心证据仍是类字节码。九位 PIN 的状态空间很小，准确复刻状态更新后穷举比试图手算 495 个初始化位更可靠。
