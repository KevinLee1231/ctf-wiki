# week4flip

## 题目简述

题目给出 `pwd.mp3` 和 `galf_si_siht_2.zip`，所有关键数据都围绕“翻转”设计：音频中的摩斯文本需要整体反转，文本文件中的二进制位串需要逐行反转后再整体反转，最终图片则是整个文件的字节序被逆置。此外，中间文件并非单个 ZIP，而是两个 ZIP 首尾拼接而成。

## 解题过程

观察 `pwd.mp3` 的长短音并按摩斯编码转换，得到：

```text
400856699300SIDROWSSAP
```

将字符串整体反转后是 `PASSWORDIS003996658004`，所以外层压缩包密码为：

```text
003996658004
```

外层 ZIP 中只有 `galf_si_siht.zip`。解密后检查文件签名，可以在偏移 `4022`（`0xFB6`）再次看到 `PK\x03\x04`：前半部分是加密 ZIP，后半部分是未加密 ZIP，而不是一个损坏后只能依赖 `binwalk` 猜测的压缩包。

后半个 ZIP 中的 `password.txt` 包含一行提示和 100 行二进制数据。第一行整体反转后为：

```text
I want to write file in binary, but something seems to be wrong?
```

其余各行先反转位序、转为字符，再把拼接出的 100 字符整体反转，得到完整提示：

```text
You are so clever that this problem is just a piece of cake for you, the password is A_pi3ce_0f_C4ke
```

因此前半个 ZIP 的密码为 `A_pi3ce_0f_C4ke`。解出 `galf_si_siht.png` 后，它的全部文件字节仍是倒序的；再次整体反转即可恢复 PNG。下面的脚本完整复现拆分、解密和恢复过程：

```python
from io import BytesIO
from pathlib import Path
from zipfile import ZipFile

source = Path("galf_si_siht_2.zip")

with ZipFile(source) as outer_zip:
    joined = outer_zip.read(
        "galf_si_siht.zip",
        pwd=b"003996658004"
    )

split = joined.find(b"PK\x03\x04", 4)
assert split == 4022

with ZipFile(BytesIO(joined[split:])) as hint_zip:
    lines = hint_zip.read("password.txt").decode().splitlines()

hint = lines[0][::-1]
message = "".join(
    chr(int(line[::-1], 2)) for line in lines[1:101]
)[::-1]
print(hint)
print(message)

with ZipFile(BytesIO(joined[:split])) as image_zip:
    reversed_png = image_zip.read(
        "galf_si_siht.png",
        pwd=b"A_pi3ce_0f_C4ke"
    )

png = reversed_png[::-1]
assert png.startswith(b"\x89PNG\r\n\x1a\n")
Path("recovered-qr.png").write_bytes(png)
```

恢复出的二维码如下：

![将倒序 PNG 的全部字节翻转后恢复的二维码，扫描结果为比赛动态 PHP 端点](0xGame2020-week4-flip-wp/recovered-qr.png)

二维码编码的是比赛期间的动态 PHP 端点，其路径指纹为 `/0xgame/flip/bd055250d3906d1f791d8e83b4396893.php`。该站点目前已经失效，仓库也没有对应后端源码，因此不把旧域名保留为可点击链接。作为可核验的历史记录，[MiaoTony 的同期 0xGame 2020 题解](https://miaotony.xyz/2020/11/05/CTF_2020_0xGame/)记载该端点当时返回：

```text
0xGame{b07906f9-f6f5-4120-9f80-01d761c8602e}
```

外链的必要信息已经写入正文，即使不打开链接也能得到最终结果；保留该链接仅用于说明动态端点响应的历史来源。

## 方法总结

完整链路是“摩斯文本整体反转 → 外层 ZIP 解密 → 按第二个 ZIP 头拆分拼接文件 → 位串逐行反转并整体反转 → 解密前半 ZIP → 图片文件全部字节反转 → 扫描二维码”。遇到貌似损坏的压缩包时，应先检查文件签名和中央目录边界；本题的双 ZIP 拼接结构可以精确拆分，无需依赖批量文件雕刻。
