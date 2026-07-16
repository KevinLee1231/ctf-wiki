# ezVBS

## 题目简述

附件 `runme.vbs` 是一个多层自修改 VBS。代码用大量 `Chr(...)` 算术表达式、可打印字符轮转和 `Execute` 隐藏下一层，并在执行后通过 `Scripting.FileSystemObject` 把新代码覆盖写回 `WScript.ScriptFullName`。

因此题目提示“不要运行”是关键：直接执行原文件虽然能进入下一层，却会丢失当前层内容。正确方法是在副本上把动态执行改为只输出，逐层静态剥离，最后逆向自定义 Base64 校验。

## 解题过程

### 1. 阻止自修改，只输出下一层

VBS 中冒号 `:` 可以在同一物理行分隔多条语句。原文件的主结构可以概括为：

```vb
aaa = Chr(...) & Chr(...) & ...
Execute(Chr(...) & Chr(...) & ...)
```

生成出的后续层还包含：

```vb
Set objFSO = CreateObject("Scripting.FileSystemObject")
strScriptPath = WScript.ScriptFullName
strTextToWrite = code
objFSO.OpenTextFile(strScriptPath, 2, True).WriteLine(strTextToWrite)
```

其中打开模式 `2` 表示覆盖写。先复制一份脚本，把当前层负责解码下一层的 `Execute` 改为 `WScript.Echo`，再运行：

```powershell
cscript //nologo runme-copy.vbs > layer1.vbs
```

对输出文件重复同样操作。某一层会出现下面的可打印字符轮转函数：

```vb
Function l(str)
    Dim i, j, k, r
    j = Len(str)
    r = ""

    For i = 1 To j
        k = Asc(Mid(str, i, 1))
        If k >= 33 And k <= 126 Then
            r = r & Chr(33 + ((k + 14) Mod 94))
        Else
            r = r & Chr(k)
        End If
    Next

    l = r
End Function
```

它在 ASCII `33..126` 的 94 个可打印字符内循环平移 14 位。继续把调用下一层的 `Execute l("...")` 改成 `WScript.Echo l("...")`，直到得到最终校验逻辑。

### 2. 识别自定义 Base64

格式化后的最终脚本如下：

```vb
str = InputBox("Enter your message:")
Base64Table = "fx6LUY5at9lnwmd3TbqzuRy+AipWHPDoXZKMFGCV2I/QjSreEsh18NJkg0v74OcB"
flag = "waZaAyNGDJ9CwLfNdzYCnyUsAJtSmLU0wqNKmLYFnyT8iyRMi5UEAMH0da8="
enc = ""

For i = 1 To Len(str) Step 3
    If i + 2 <= Len(str) Then
        bitBlock = Asc(Mid(str, i, 1)) * 256 * 256 _
                 + Asc(Mid(str, i + 1, 1)) * 256 _
                 + Asc(Mid(str, i + 2, 1))
        c1 = Mid(Base64Table, Int(bitBlock \ (64 * 64 * 64)) Mod 64 + 1, 1)
        c2 = Mid(Base64Table, Int(bitBlock \ (64 * 64)) Mod 64 + 1, 1)
        c3 = Mid(Base64Table, Int(bitBlock \ 64) Mod 64 + 1, 1)
        c4 = Mid(Base64Table, Int(bitBlock) Mod 64 + 1, 1)
        enc = enc & c1 & c2 & c3 & c4
    ElseIf i + 1 <= Len(str) Then
        bitBlock = Asc(Mid(str, i, 1)) * 256 * 256 _
                 + Asc(Mid(str, i + 1, 1)) * 256
        c1 = Mid(Base64Table, Int(bitBlock \ (64 * 64 * 64)) Mod 64 + 1, 1)
        c2 = Mid(Base64Table, Int(bitBlock \ (64 * 64)) Mod 64 + 1, 1)
        c3 = Mid(Base64Table, Int(bitBlock \ 64) Mod 64 + 1, 1)
        enc = enc & c1 & c2 & c3 & "="
    Else
        bitBlock = Asc(Mid(str, i, 1)) * 256 * 256
        c1 = Mid(Base64Table, Int(bitBlock \ (64 * 64 * 64)) Mod 64 + 1, 1)
        c2 = Mid(Base64Table, Int(bitBlock \ (64 * 64)) Mod 64 + 1, 1)
        enc = enc & c1 & c2 & "=="
    End If
Next

If enc = flag Then
    MsgBox "Your flag is correct!"
Else
    MsgBox "Your flag is incorrect!"
End If
```

分组、移位和 `=` 填充都与标准 Base64 相同，只有索引到字符的 64 字符表被替换。把密文中的自定义字符按相同索引翻译回标准字母表，再调用普通 Base64 解码即可：

```python
import base64


custom = "fx6LUY5at9lnwmd3TbqzuRy+AipWHPDoXZKMFGCV2I/QjSreEsh18NJkg0v74OcB"
standard = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
ciphertext = "waZaAyNGDJ9CwLfNdzYCnyUsAJtSmLU0wqNKmLYFnyT8iyRMi5UEAMH0da8="

assert len(custom) == 64
translation = str.maketrans(custom, standard)
plaintext = base64.b64decode(ciphertext.translate(translation)).decode()
print(plaintext)
```

输出为：

```text
0xGame{bf00591f-a1cb-4191-b41d-d4eecda0b798}
```

## 方法总结

本题由“算术构造字符 → `Execute` 动态执行 → 可打印字符轮转 → 覆盖自身 → 自定义 Base64”组成。危险点不在最终编码，而在前几层具有真实副作用：若直接执行原脚本，旧层会被覆盖，证据链随之丢失。

处理自修改脚本时应先备份，并把执行原语替换成输出原语，使每层结果单独落盘。到最终校验后，要从分组宽度、位运算和填充规则识别算法，而不是只看变量名；自定义 Base64 通常只需按索引还原字母表，不必重写整个解码器。
