# easyApp

## 题目简述

附件 `easyApp.apk` 是一个 Android 输入校验程序，入口为 `com.example.easyapp.MainActivity`。应用要求输入长度恰好为 42，并把字符串拆成两段：前 26 字符做标准 Base64 比较，后 16 字符转为十六进制后交给动态加载的 `Secret.check` 校验。

第二阶段代码没有放在主 `classes.dex` 中，而是藏在 `assets/dex.zip` 的 `dex.bin` 内。主程序把该资源复制到缓存目录，通过 `DexClassLoader` 加载 `com.example.easyapp.Secret`，再反射调用静态方法 `check(String)`。

## 解题过程

### 1. 还原 `MainActivity` 的分段逻辑

从 `AndroidManifest.xml` 定位到 `MainActivity` 后，按钮回调的核心逻辑可以整理为：

```java
String input = editText.getText().toString();

if (input.length() != 42) {
    showToast("Length wrong!");
    return;
}

String first = input.substring(0, 26);
String encoded = Base64.encodeToString(
    first.getBytes(StandardCharsets.UTF_8),
    Base64.DEFAULT
).trim();

byte[] second = input.substring(26).getBytes(StandardCharsets.UTF_8);

if (encoded.equals("MHhHYW1le0RvX3kwdV9sMHYzX2FuZHIwMWQ=")) {
    Method check = new DexClassLoader(
        dexZip.getAbsolutePath(),
        getCacheDir().getAbsolutePath(),
        null,
        getClassLoader()
    ).loadClass("com.example.easyapp.Secret")
     .getDeclaredMethod("check", String.class);

    boolean ok = (Boolean) check.invoke(null, toHexString(second));
}
```

先解码 Base64 常量：

```python
import base64


encoded = "MHhHYW1le0RvX3kwdV9sMHYzX2FuZHIwMWQ="
first = base64.b64decode(encoded).decode()
print(first, len(first))
```

输出：

```text
0xGame{Do_y0u_l0v3_andr01d 26
```

这里是 `andr01d`，末尾依次为数字 `0`、数字 `1`，不能写成 `andr0id`。

### 2. 提取并分析动态 Dex

从 APK 中取出 `assets/dex.zip`，再提取其中的 `dex.bin`。文件本身是 DEX 字节码，反编译后只有类 `com.example.easyapp.Secret`。`check` 接收后半段的 32 位十六进制字符串，并构造三个互相重叠的 64 位大整数：

```java
BigInteger a = new BigInteger(hex.substring(0, 16), 16);
BigInteger b = new BigInteger(hex.substring(8, 24), 16);
BigInteger c = new BigInteger(hex.substring(16, 32), 16);

return b.add(a.multiply(BigInteger.valueOf(3)))
            .subtract(new BigInteger("27454419028250566601"))
            .equals(BigInteger.ZERO)
    && c.multiply(BigInteger.valueOf(2))
            .subtract(b.multiply(BigInteger.valueOf(5)))
            .add(new BigInteger("20616666104378640363"))
            .equals(BigInteger.ZERO)
    && a.add(c.multiply(BigInteger.valueOf(4)))
            .subtract(new BigInteger("1dce62be9f0fa2f6c", 16))
            .equals(BigInteger.ZERO);
```

对应方程为：

```text
3a + b = 27454419028250566601
2c - 5b + 20616666104378640363 = 0
a + 4c = 0x1dce62be9f0fa2f6c
```

注意切片不是三个独立块：`a` 覆盖字节 0～7，`b` 覆盖字节 4～11，`c` 覆盖字节 8～15。解出三个整数后，`a || c` 正好覆盖完整 16 字节，而 `b` 应等于拼接结果中间的 8 字节。

### 3. 求解并拼接完整 flag

使用 Z3 求解，并额外验证重叠窗口的一致性：

```python
import base64
from z3 import Ints, Solver, sat


first = base64.b64decode(
    "MHhHYW1le0RvX3kwdV9sMHYzX2FuZHIwMWQ="
).decode()

a, b, c = Ints("a b c")
solver = Solver()
solver.add(3 * a + b == 27454419028250566601)
solver.add(2 * c - 5 * b + 20616666104378640363 == 0)
solver.add(a + 4 * c == int("1dce62be9f0fa2f6c", 16))

assert solver.check() == sat
model = solver.model()

a_hex = f"{model[a].as_long():016x}"
b_hex = f"{model[b].as_long():016x}"
c_hex = f"{model[c].as_long():016x}"
tail_hex = a_hex + c_hex

assert tail_hex[8:24] == b_hex
second = bytes.fromhex(tail_hex).decode()
flag = first + second

print(a_hex)
print(b_hex)
print(c_hex)
print(second)
print(flag)
```

输出：

```text
5f346e645f646578
5f6465785f6c6f61
5f6c6f616465727d
_4nd_dex_loader}
0xGame{Do_y0u_l0v3_andr01d_4nd_dex_loader}
```

## 方法总结

本题的校验链为“长度 42 → 前 26 字符 Base64 → 后 16 字符转十六进制 → 动态 Dex 反射调用 → 三元一次方程组”。主 Dex 只负责调度，真正的第二阶段约束位于 assets 中，因此看到 `getAssets().open`、`DexClassLoader`、`loadClass` 和 `getDeclaredMethod` 时，应继续提取被加载的文件，不能停在主程序。

解方程之外还要理解数据切片。三个整数是相互重叠的窗口，而不是顺序排列的 24 字节；用 `a` 与 `c` 拼接、再用 `b` 校验中间重叠部分，才能无歧义地恢复原始 16 字节字符串。
