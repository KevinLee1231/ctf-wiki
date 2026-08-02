# N1CTF 2022 - Desktop-Apps

## 题目简述

附件是一个 Electron 桌面应用。JavaScript 源码没有直接出现，核心逻辑被编译成与特定 V8 版本绑定的 `.jsc` 字节码；同时 `app.asar` 被加入垃圾数据，常规 `asar extract` 无法直接解包。

目标是恢复 Electron 的加载链，反序列化并反汇编 V8 字节码，再还原 renderer 中对校验常量的二次变换，最终解出一个 $7\times7$ 矩阵方程。

## 解题过程

### 修复 ASAR 并定位 JSC

Electron 应用的资源通常位于 `resources/app.asar`。虽然本题破坏了归档使整体解包失败，但单文件提取仍然可用，也可以按 ASAR 头部记录重建全部有效文件。

入口 `main.js` 加载 `main.jsc`，preload 脚本继续加载 `preload.jsc`，renderer 则涉及：

```text
renderer.jsc
a783a16e1a0b62e833626fbb895b8b16026ecfa5f7db3cfe7f10f11afb43.jsc
```

应用中可以确定版本：

```text
Electron 22.0.0-alpha.3
V8 10.8.79-electron.0
```

V8 字节码格式随版本变化，因此应选择尽可能接近的 V8 10.8.79，而不是用任意新版 Node.js 强行加载。

### 给 d8 增加 JSC 反序列化入口

`.jsc` 来自 `v8::internal::CodeSerializer::Serialize`。对应的读取入口是 `CodeSerializer::Deserialize`，其返回的 `SharedFunctionInfo` 可通过 `GetActiveBytecodeArray()` 取得字节码数组，再调用 `BytecodeArray::Disassemble()`。

编译 d8 时启用反汇编和对象打印：

```bash
tools/dev/v8gen.py x64.release -- \
  v8_enable_disassembler=true \
  v8_enable_object_print=true
ninja -C out.gn/x64.release d8
```

随后在 `d8.cc` 中增加一个 `loadjsc()`：读取文件为 `AlignedCachedData`，调用 `CodeSerializer::Deserialize`，递归打印主函数和常量池内嵌套 `SharedFunctionInfo` 的字节码。还要在 `deserializer.cc` 与 `code-serializer.cc` 中绕过 `source_hash`、`flags_hash` 的一致性检查，否则没有原始源码和完全相同的启动参数时会拒绝反序列化。

数组字面量在常量池中表现为 `ArrayBoilerplateDescription`，默认反汇编不会展开具体元素。因此加载器还需要递归输出它的 `constant_elements`。各字节码指令的语义可对照同版本 V8 的 [interpreter-generator.cc](https://github.com/v8/v8/blob/10.8.79/src/interpreter/interpreter-generator.cc)；本题真正用到的是数组、循环、属性访问、Proxy 和普通算术，没有必要还原所有 V8 指令。

### 还原校验逻辑与 Proxy

哈希命名的 JSC 中，`checkflag` 先要求输入长度为 49，再按行转换为 $7\times7$ 字符码矩阵 $A$，最后检查：

$$
A B = Y
$$

但反汇编得到的原始 `_B` 和 `_Y` 不能直接代入，因为 `renderer.jsc` 用 `Proxy` 拦截每一行的读取。

读取 `_B` 时，每个元素执行：

```javascript
value ^= 0xbeef;
value -= 0xdead;
```

读取 `_Y` 时，先把每一行反转，再连续执行两次：

```javascript
value = mask & ~value | ~mask & value;
```

该表达式等价于 `mask ^ value`。两个掩码分别为 `0xfeebdaed` 和 `0xdeadbeef`，并遵循 JavaScript 32 位按位运算语义。完成这些变换后，实际系数矩阵 $B$ 为：

```python
B = [
    [78,117,49,76,84,51,52],
    [77,78,85,49,76,84,51],
    [52,77,78,117,105,76,84],
    [51,52,77,78,117,49,76],
    [116,51,52,77,78,117,108],
    [76,84,51,52,77,78,53],
    [49,76,84,51,52,77,78],
]
```

目标矩阵 $Y$ 为：

```python
Y = [
    [41022,45573,39758,40677,48362,44403,42084],
    [41955,43622,39931,41526,47256,46827,43586],
    [38709,44604,40779,42848,49939,42721,41448],
    [48675,49275,44576,45757,55294,51436,47595],
    [43099,43111,40162,41324,47815,47359,42950],
    [48531,51268,46243,45685,56210,51027,47819],
    [41294,49458,43359,46491,53162,45199,44838],
]
```

### 解矩阵并验证

由 $AB=Y$ 得 $A=YB^{-1}$。使用 NumPy 求解后四舍五入为整数，并重新相乘验证，避免浮点误差产生错误字符：

```python
import numpy as np

B = np.array([
    [78,117,49,76,84,51,52],
    [77,78,85,49,76,84,51],
    [52,77,78,117,105,76,84],
    [51,52,77,78,117,49,76],
    [116,51,52,77,78,117,108],
    [76,84,51,52,77,78,53],
    [49,76,84,51,52,77,78],
])
Y = np.array([
    [41022,45573,39758,40677,48362,44403,42084],
    [41955,43622,39931,41526,47256,46827,43586],
    [38709,44604,40779,42848,49939,42721,41448],
    [48675,49275,44576,45757,55294,51436,47595],
    [43099,43111,40162,41324,47815,47359,42950],
    [48531,51268,46243,45685,56210,51027,47819],
    [41294,49458,43359,46491,53162,45199,44838],
])

A = np.rint(Y @ np.linalg.inv(B)).astype(int)
assert np.array_equal(A @ B, Y)
print("".join(chr(x) for x in A.flat))
```

本地复算输出：

```text
N1CTF{y3Ea4h_tHIs_1S_NlDeskT0p_4pPS_n0w_enj0y_1T}
```

## 方法总结

这道题有三层障碍：损坏的 ASAR、版本绑定的 V8 序列化字节码，以及 renderer 对常量的运行时 Proxy 变换。只反汇编校验模块会得到错误矩阵，必须同时恢复加载它的 renderer。处理 V8 cache 时，版本、hash 检查和常量池数组展开缺一不可；最终得到线性关系后，应把逆矩阵结果重新代回原式验证，而不是只相信浮点输出。
