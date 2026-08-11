# A R C

## 题目简述

题目把线索串在图片、换表 Base58、视频、变位移 ROT47、MSU StegoVideo 和维吉尼亚密码之间。最关键的凭据藏在视频隐写载荷中；其余编码负责给出压缩包密码、隐写密码和最终路径。因此按决定性障碍归入 Stego，而不是沿用官方总 PDF 的 Misc 外壳。

## 解题过程

解压后首先得到 `8558.png`。文件名应拆成 85 与 58：图片中的字符采用题目提供的特殊字体，需要先逐字辨认，抄出的 Base85 文本为：

```text
BK0ICG]Qr*88_$gC,'-j2+KH86?Q"%928;LG@O*!Am0+`;E7iV2agSE<c'U;6Yg^#H?!YBAQ]
```

Base85 解码后得到一条长度为 58 的字符串：

```text
h8btxsWpHnJEj1aL5G3gBuMTKNPAwcF4fZodR9XQ7DSUVm2yCkr6zqiveY
```

它不是普通密文，而是被替换过的 Bilibili BV 编码表。特殊字体在这里有双重作用：既是抄写图片字符的对照表，也暗示后续要处理一张编码 `table`。辨认时尤其要区分 `i`/`j` 与大写 `I`/小写 `l`。

使用该换表版本编码十进制 `10001540`，得到压缩包密码。官方题解给出的完整编码代码如下：

```python
table = "h8btxsWpHnJEj1aL5G3gBuMTKNPAwcF4fZodR9XQ7DSUVm2yCkr6zqiveY"
positions = [11, 10, 3, 8, 4, 6]
xor_value = 177451812
add_value = 8728348608


def bv_encode(value):
    value = (value ^ xor_value) + add_value
    result = list("BV1  4 1 7  ")
    for index in range(6):
        result[positions[index]] = table[value // 58**index % 58]
    return "".join(result)


print(bv_encode(10001540))
```

输出为 `BV17f411J77h`，用它解压可得到 `arc.mkv` 与 `fragment.txt`。视频里出现问题：

```text
What is the answer to life, the universe and everything?
```

答案是 42。视频末尾和 `fragment.txt` 都有一段落在可打印 ASCII 范围内的变位移 ROT47；字符集合仍是 ASCII 33 至 126，但位移改为 42。空格保持不变，解码函数为：

```python
def rotated_ascii_decode(text):
    result = []
    for char in text:
        code = ord(char)
        if 33 <= code <= 126:
            result.append(chr(33 + (code - 33 + 42) % 94))
        else:
            result.append(char)
    return "".join(result)
```

例如视频第一行 `#)+F7IIMEH:?Injiikffi` 解码为：

```text
MSUpasswordis:6557225
```

`fragment.txt` 的较长文本用于词频分析并确认相同变换；解码后的第一行明确说明它不含 flag，只是帮助识别编码。密码 `6557225` 则用于视频隐写提取。题目使用 [MSU StegoVideo 1.0](https://www.compression.ru/video/stego_video/index_en.html)：把插件放入 VirtualDub 的 `plugins32` 目录，导入 `arc.mkv`，依次选择 `Video → Filters → Add → MSU StegoVideo 1.0`，选中 `Extract file from video`，填写密码与输出路径，再保存处理后的视频以触发提取。

隐藏文本给出三行凭据：

```text
arc.hgame2021.cf
Hikari
Tairitsu
```

用后两行登录第一行所示站点后，还需要处理第二条提示：`For line2, Liki has told you what it is, and Akira is necessary to do it.`。Week1 中与 Liki 相关的算法是维吉尼亚密码，因此以 `Akira` 为循环密钥解密：

```text
密文：pwbvmpoakiscqdobil
密钥：Akira
明文：pmtempestissimobyd
```

明文不是表单输入，而是 URL 路径。访问 `/pmtempestissimobyd` 后得到：

```text
hgame{Y0u_Find_Pur3_Mem0ry}
```

## 方法总结

长链题要让每条线索承担明确角色：`8558` 指向 Base85 与 Base58，换表 BV 编码生成压缩包密码，视频问题给出 ROT 位移 42，ROT 解码得到 MSU 隐写密码，视频载荷给出站点登录凭据，最后再用提示指定的维吉尼亚密钥恢复路径。外部工具链接只保留了实际提取载荷所需的 MSU StegoVideo 页面；BV 原理、字符表和密码生成逻辑已经完整写入正文，无需依赖通用说明外链。
