# guess-again

## 题目简述

附件是启用宏的 Excel 工作簿 `chall.xlsm`，表面上要求在文本框中输入内容并按下按钮。flag 校验并不在单元格公式中，而是在 VBA 宏 `CheckFlag` 内；需要提取宏代码、辨别未调用的诱饵函数，再把十进制字符码还原为 ASCII。

## 解题过程

不要直接启用来源不明工作簿的宏。用 `olevba chall.xlsm` 静态提取 `xl/vbaProject.bin`，可以在 `VBA/Module1` 中找到按钮绑定的 `CheckFlag`。其有效逻辑可整理为：

```vba
guess = ActiveSheet.Shapes("TextBox 1").TextFrame2.TextRange.Text

If Left(guess, 6) <> "tjctf{" Or Right(guess, 1) <> "}" Then
    MsgBox "Flag must start with tjctf{ and end with }"
    Exit Sub
End If

inner = Mid(guess, 7, Len(guess) - 7)
expectedCodes = Array(98, 117, 116, 95, 99, 52, 110, 95, 49, _
                      116, 95, 114, 117, 110, 95, 100, 48, 48, 109)

For i = 1 To Len(inner)
    If Asc(Mid(inner, i, 1)) <> expectedCodes(i - 1) Then
        MsgBox "Incorrect"
        Exit Sub
    End If
Next i
```

数组共有 19 项，逐项转成字符即可：

```python
codes = [
    98, 117, 116, 95, 99, 52, 110, 95, 49, 116,
    95, 114, 117, 110, 95, 100, 48, 48, 109,
]
inner = "".join(map(chr, codes))
print(f"tjctf{{{inner}}}")
```

输出为：

```text
tjctf{but_c4n_1t_run_d00m}
```

宏工程里还能看到 `Module2.Validate` 中的 `tjctf{fake_flag}`，以及一个未被 `CheckFlag` 调用的 `check` 辅助函数。它们都不在按钮触发的实际调用链上，因此不能只按字符串搜索取第一个看似 flag 的结果。

## 方法总结

- 核心技巧：静态提取 VBA，沿按钮入口函数追踪真实校验链，并把字符码数组恢复为字符串。
- 识别信号：`.xlsm`、表单按钮、文本框输入和“按下按钮”提示都应优先检查宏工程。
- 复用要点：恶意 Office 宏可能在打开或启用内容时执行；应先在隔离环境静态分析，并用调用关系排除假 flag 和死代码。
