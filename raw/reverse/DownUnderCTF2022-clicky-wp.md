# DownUnderCTF 2022 clicky Writeup

## 题目简述

题目给出一个 .NET 6 Windows Forms 单文件程序 `ClickMe.exe`。运行后需要不断点击会移动、缩小并加速的图片控件，但 flag 的各段实际上藏在托管程序集的字段和界面标签中。目标是从单文件 bundle 中提取程序集，并还原程序拼接 flag 的过程。

## 解题过程

直接把主文件交给普通 .NET 反编译器会提示它不是托管 PE，因为文件开头是 .NET 单文件宿主。检查 bundle 内部可以找到第二个 `MZ`，其偏移为 `0x24600`（148992）；从这里切出大小 54272 字节的托管映像后，即可正常反编译 `Form1.cs`。

点击处理函数按控件尺寸切换状态：图片从 $100\times100$ 变为 $75\times75$，再变为 $10\times10$；计时器间隔依次由 1420 ms 降至 420 ms 和 69 ms。第三次成功点击后，程序不再依赖游戏状态，而是直接拼接若干解码结果。

核心辅助函数只有两种变换：

```csharp
string Unscramble(string s) {
    return Base64Decode(Base64Decode(s));
}

string Random_function(string s) {
    return HexDecode(s);
}
```

三个放在窗口可视区域之外的标签内容为：

```text
576B64736131677A62485A6B55543039
57444E57656C70574F44303D
57565935565646575453383D
```

先十六进制解码，再按 `Unscramble` 做两次 Base64 解码，它们分别得到 `did_you`、`_use_` 和 `a_TAS?`。其余字段解码得到 `DUCTF`、`{` 和 `}`。需要特别注意，代码中的 `_ZGVhZGIzM2ZjYWZl` 是直接追加的字面量，并没有再进入 Base64 解码函数。最终拼接顺序为：

```text
DUCTF + { + did_you + _use_ + a_TAS? + _ZGVhZGIzM2ZjYWZl + }
```

因此二进制实际弹窗显示的是：

```text
DUCTF{did_you_use_a_TAS?_ZGVhZGIzM2ZjYWZl}
```

但公开仓库的 `ctfcli.yaml` 把比赛平台正式验收值登记为末字符不同的大写 `I`：

```text
DUCTF{did_you_use_a_TAS?_ZGVhZGIzM2ZjYWZI}
```

两者并不等价：`ZGVhZGIzM2ZjYWZl` 解码为 `deadb33fcafe`，而 `ZGVhZGIzM2ZjYWZI` 解码为 `deadb33fcafH`。复现程序行为时应保留前者，提交或核对赛事元数据时应使用后者。

## 方法总结

题目的主要障碍是 .NET 单文件 bundle，而不是点击操作本身。识别原生宿主中的嵌入式托管 PE 后，界面事件和隐藏标签都能直接恢复。处理多层编码时应严格按调用顺序逐段解码，并区分“作为解码函数参数的字符串”和“代码直接拼接的字符串”，否则很容易把 flag 中本应保留的 Base64 外观片段误解码。本题还说明二进制输出与赛事验收元数据可能不一致，严谨归档必须把两种证据及其适用场景分别注明。
