# DownUnderCTF 2020 - im getting emotional

## 题目简述

附件是启用宏的 Word 文档 `.docm`。`Document_Open` 会弹出输入框，生成一个硬编码目标字符串，再用 `StrComp` 比较用户输入。VBA 经变量改名、算术表达式和文档变量中的 Base64 数据混淆。为避免触发自动宏，分析过程应静态提取 VBA，不在 Word 中直接打开并启用内容。

## 解题过程

使用 `olevba` 的只读提取功能可以定位自动执行入口和关键比较：

```bash
python -m oletools.olevba --decode --reveal "important information !!! .docm"
```

整理混淆表达式后，主流程是：

```vb
Private Sub Document_Open()
    CkJufCjNKA = InputBox(...)
    Dim QVEuMQzkhLSzj As String
    QVEuMQzkhLSzj = RoUVEJlEsXYg(Array(132), 80) _
        & RoUVEJlEsXYg(Array(234, 97, 188, 8, 210, 95, 130, 46, _
          193, 68, 97, 177, 197, 236, 215, 199, 163, 97, 19, 253), 81)
    result = StrComp(CkJufCjNKA, QVEuMQzkhLSzj)
End Sub
```

解码函数先对 `ActiveDocument.Variables("HlAcnXbyvOeFkaEM")` 中的 Base64 字符串解码，再将指定偏移处的每个字节与数组元素异或：

```vb
zMKdWGVWKsxoD = ZXfZBmatULAgs(ActiveDocument.Variables("HlAcnXbyvOeFkaEM"))
For i = LBound(values) To UBound(values)
    output = output & Chr(zMKdWGVWKsxoD(i + offset) Xor values(i))
Next
```

文档变量的内容为：

```text
63CA4dREjhYqoWoTMH/5o1SCPKHs2WwJD+MaZom0Zw/wI0pkF787BQ4Ofrli1e/Nmys0UQNi0y8ZIuQdudlzHQsUJVvkCxKpy7rVIRZPlq/AvyLoTqkS402zdD7Z8ZTntc4AfYA6y2LShV6hNq3R+db29xlQiChHFcnbwT2i5enD8F4jHesJs02twg==
```

可以用等价的最小脚本复现这一步，而无需执行宏：

```python
from base64 import b64decode

blob = b64decode("63CA4dREjhYqoWoTMH/5o1SCPKHs2WwJD+MaZom0Zw/wI0pkF787BQ4Ofrli1e/Nmys0UQNi0y8ZIuQdudlzHQsUJVvkCxKpy7rVIRZPlq/AvyLoTqkS402zdD7Z8ZTntc4AfYA6y2LShV6hNq3R+db29xlQiChHFcnbwT2i5enD8F4jHesJs02twg==")
parts = [([132], 80),
         ([234, 97, 188, 8, 210, 95, 130, 46, 193, 68,
           97, 177, 197, 236, 215, 199, 163, 97, 19, 253], 81)]

answer = ''.join(
    chr(blob[offset + i] ^ value)
    for values, offset in parts
    for i, value in enumerate(values)
)
print(answer)
```

输出为：

```text
DUCTF{Macr0_h4x0rman}
```

官方动态解法是在 VBA 编辑器中对 `StrComp` 下断点或加监视表达式，读取第二个参数；但静态提取既能得到相同结果，也避免运行潜在恶意宏。原题中的调试器截图只展示这段文本信息，因此不作为有独立视觉价值的图片保留。

## 方法总结

分析宏文档时，首先识别 `Document_Open` 等自动执行点，并优先静态提取。混淆变量名并不重要，真正的数据流是“文档变量 Base64 解码 → 指定偏移逐字节异或 → `StrComp` 比较”。将该数据流缩成一个无副作用的解码器，就能安全且可复现地获得答案。
