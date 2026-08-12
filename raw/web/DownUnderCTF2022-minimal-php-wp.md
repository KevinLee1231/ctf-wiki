# DownUnderCTF 2022 minimal-php Writeup

## 题目简述

服务端只有一行 PHP：

```php
<?php file($_POST[0]);
```

`file()` 会读取客户端指定的资源，却既不返回内容也不显示错误。flag 位于 `/flag`。题目要求组合 `php://filter` 的长度放大、`dechunk` 特性和字符重排过滤器，把“是否触发 PHP 内存上限”变成盲读文件的错误 oracle。

## 解题过程

请求形式为：

```python
import requests

def request_filter(chain):
    r = requests.post('http://target/index.php', data={
        '0': f'php://filter/{chain}/resource=/flag',
    })
    return r.status_code == 500
```

### 建立空串 oracle

`convert.iconv.L1.UCS-4LE` 会把每个输入字节扩成 4 字节。连续应用足够多次时，非空流会指数增长并超过 `memory_limit`，产生 HTTP 500；空流则不会。由于实际内存上限和 PHP 开销会变化，先逐步增加过滤器数量，测出刚好溢出的基线，再把“基线减一”作为后续放大尾链。

`dechunk` 过滤器在无换行输入上还有一个可利用的分支：若首字符属于 `0-9A-Fa-f`，它会把输入当成不完整的块长度并消去流；否则保留内容。将它放在放大链之前，就能判断当前首字符是否属于十六进制字符集：

```text
待测变换 | dechunk | 多次 convert.iconv.L1.UCS-4LE
```

没有 500 表示 `dechunk` 清空了流，出现 500 表示流仍非空。

### 把任意位置移到开头

后续字符定位要求输入只含字母数字，并需要以 `==` 结尾。先对 `/flag` 连续做两次 Base64 编码；若填充不合适，就在两次编码之间加入一次或两次 `convert.iconv..CSISO2022KR`，用固定前缀改变长度，直到双重 Base64 以两个等号结尾。

两个过滤器提供基本换位操作：

```text
convert.iconv.CSUNICODE.UCS-2BE       : abcdefgh -> badcfehg
convert.iconv.UCS-4LE.10646-1:1993   : abcdefgh -> dcbahgfe
```

另一个关键链先把结尾的 `==` 通过 quoted-printable 与 UTF-7 变成可控填充，再由 `convert.iconv.CSUNICODE.CSUNICODE` 在开头加入 BOM。四字节翻转后，BOM 落到中间；`base64-decode|base64-encode` 会丢弃无效高字节，第二次四字节翻转便把后续两个字符移到流首。官方 solver 将这段操作记为 `flip`：

```python
flip = (
    'convert.quoted-printable-encode|convert.quoted-printable-encode|'
    'convert.iconv.L1.utf7|convert.iconv.L1.utf7|'
    'convert.iconv.L1.utf7|convert.iconv.L1.utf7|'
    'convert.iconv.CSUNICODE.CSUNICODE|'
    'convert.iconv.UCS-4LE.10646-1:1993|'
    'convert.base64-decode|convert.base64-encode'
)
```

重复组合 `flip`、两字节翻转和四字节翻转，就能为第 $n$ 个字符生成一条过滤链，使该字符移动到首位。

### 识别字符并还原 flag

对首字符先直接使用 `dechunk`，再依次尝试 `string.rot13`、`string.tolower`、`string.toupper` 以及若干 iconv 编码映射。每条映射都把一个小字符集合送入 `0-9A-Fa-f`，由空串 oracle 给出一位结果；组合这些结果形成决策树，就能区分所有 Base64 字母。数字不易被 iconv 单独区分时，再做一次 Base64 编码，检查其前两个字符即可分成 `0-3`、`4-7`、`8-9` 并确定具体数字。

逐位取得双重 Base64 文本后，先移除可能由 `CSISO2022KR` 加入的 `\x1b$)C`，再解码两次：

```python
from base64 import b64decode

stage1 = b64decode(leaked_text + '=' * 4)
stage1 = stage1.replace(b'\x1b$)C', b'')
flag = b64decode(stage1)
print(flag.decode())
```

最终得到：

```text
DUCTF{i_love_minimal_php...<3...how_about_you?}
```

## 方法总结

本题没有直接输出，完整利用由三个可组合原语构成：内存放大把空串变成状态码 oracle，`dechunk` 判断首字符是否落在十六进制集合，iconv/Base64 过滤链负责移动和映射任意字符。真正困难的是保持每次变换后的长度、字符集和 `==` 填充不变量；先校准放大阈值，再独立验证定位链和分类链，能避免把网络波动误当作字符结果。
