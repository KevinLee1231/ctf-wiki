# native app

## 题目简述

题目是一个 Flutter 原生应用。界面包含文本框和“提交”按钮，但校验并不是一次点击完成：输入框的提交回调先预处理文本，短按按钮执行 RC4，长按按钮才把密文与内置数组比较。

Flutter release 应用的 Dart 代码经过 AOT 编译，主要逻辑位于 APK 对应 ABI 目录中的 `libapp.so`，不能按普通 Java/Kotlin APK 只看 `classes.dex`。目标是恢复 Dart 回调和常量，逆出能够显示 `true` 的输入。

## 解题过程

### 1. 从 Flutter AOT 中定位回调

识别 APK 中的 Flutter engine 和 `libapp.so` 后，可以用两条路线恢复逻辑：

- 用 blutter 解析 Flutter AOT snapshot 的类、函数、对象池和常量，再围绕界面字符串查找引用；
- 按作者的方法编译一个结构相近的 AOT snapshot，与题目 `libapp.so` 做 BinDiff，再沿按钮与文本框回调还原业务函数。

无论使用哪种工具，定位锚点都相同：界面字符串 `Chall`、`提交`、`submitted:`、`false`、`true`，以及 `GestureDetector` 的 `onTap`、`onLongPress` 和 `TextField` 的 `onSubmitted`。工具只是帮助恢复 AOT 结构，最后仍要核对回调间的数据流。

仓库公开的 Dart 源码确认了四个状态量：

```dart
String _inputText = '';
var inputByte = [0];
var encByte = [0];
var key = [140, 136, 210, 238, 167, 102, 222, 38];
var cmp = [
  184, 132, 137, 215, 146, 65, 86, 157, 123, 100, 179, 131,
  112, 170, 97, 210, 163, 179, 17, 171, 245, 30, 194, 144,
  37, 41, 235, 121, 146, 210, 174, 92, 204, 22
];
```

### 2. 还原真实交互顺序

文本变化和文本提交是两个不同事件：

```dart
void _onChanged(String value) {
  setState(() {
    _outputText = value;
  });
}

void _onSubmit(String value) {
  _inputText = value;
  inputByte = utf8.encode(value);
  for (var i = 0; i < inputByte.length; i++) {
    inputByte[i] ^= 0xff;
  }
}
```

仅在文本框中键入内容只会触发 `_onChanged()`，不会更新待加密的 `inputByte`。按键盘的提交/完成键触发 `_onSubmit()` 后，程序才将输入做 UTF-8 编码，并逐字节异或 `0xff`。

短按按钮执行：

```dart
void _onTap() {
  encByte = rc4Encrypt(inputByte, key);
  setState(() {
    _outputText = 'submitted:$_inputText';
  });
}
```

长按同一按钮则只比较当前 `encByte` 与 `cmp`：

```dart
void _onLongPressed() {
  if (cmp.length != encByte.length) return;
  for (int i = 0; i < cmp.length; i++) {
    if (cmp[i] != encByte[i]) {
      setState(() { _outputText = 'false'; });
      return;
    }
  }
  setState(() { _outputText = 'true'; });
}
```

所以验证顺序必须是“文本框提交 → 短按加密 → 长按比较”。这是 UI 逻辑的一部分，不能只从 `cmp` 数组推出答案后忽略操作顺序。

### 3. 利用 RC4 对称性解密

设原始 UTF-8 字节为 $P_i$，异或预处理后为 $P_i \oplus 0xff$，RC4 密钥流为 $Z_i$，比较数组为 $C_i$，则：

$$
C_i = P_i \oplus 0xff \oplus Z_i
$$

因此使用相同 key 对 `cmp` 再执行一次 RC4，就会消去密钥流，最后再异或 `0xff` 得到原文：

```python
def rc4(data, key):
    s = list(range(256))
    j = 0
    for i in range(256):
        j = (j + s[i] + key[i % len(key)]) & 0xff
        s[i], s[j] = s[j], s[i]

    i = j = 0
    out = []
    for value in data:
        i = (i + 1) & 0xff
        j = (j + s[i]) & 0xff
        s[i], s[j] = s[j], s[i]
        out.append(value ^ s[(s[i] + s[j]) & 0xff])
    return bytes(out)

key = [140, 136, 210, 238, 167, 102, 222, 38]
cmp = [
    184, 132, 137, 215, 146, 65, 86, 157, 123, 100, 179, 131,
    112, 170, 97, 210, 163, 179, 17, 171, 245, 30, 194, 144,
    37, 41, 235, 121, 146, 210, 174, 92, 204, 22,
]

decoded = rc4(cmp, key)
answer = bytes(value ^ 0xff for value in decoded)
print(answer.decode())
```

输出为：

```text
Iu2xpwXLAK734btEt9kXIhfpRgTlu6KuI0
```

将该字符串输入文本框，先提交输入，再短按和长按按钮，界面会显示 `true`。仓库只证明这是应用接受的正确输入，没有给出额外的 `ACTF{...}` 包装规则，因此不应自行改写答案格式。

## 方法总结

本题的逆向难点有两层：先从 Flutter AOT 而不是 DEX 中恢复 Dart 业务逻辑，再正确理解多个 UI 回调之间的状态依赖。算法本身是标准 RC4，加密前多了一次逐字节异或 `0xff`，利用 RC4 对称性即可直接反解。

分析 GUI 题时不能只盯住最终比较函数，还应追踪输入何时写入、转换何时发生、校验由哪个事件触发。恢复出来的常量、回调顺序和本地求解输出三者互相印证后，结论才完整。
