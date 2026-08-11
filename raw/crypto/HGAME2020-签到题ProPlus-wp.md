# 签到题 ProPlus

## 题目简述

附件由加密 ZIP、口令提示和一段 Ook 文本组成。完整链条依次为三栏栅栏密码、位移为 5 的凯撒密码、Ook、Base32、Base64、PNG 二维码；各层都是可逆编码或古典密码，因此归入 Crypto。

## 解题过程

Password.txt 末尾给出处理顺序：先用三栏栅栏密码，再用位移为 5 的凯撒密码。

```text
Three fences first, Five Caesar next. English sentence first, zip password next.
```

两层解密后，第一行是一句英文，第二行是 ZIP 密码。

```text
EAVMUBAQHQMVEPDT
```

用该口令解开 `OK.zip` 后得到 `OK.txt`。文件开头的 `data:text;ook` 明确指出主体是 Ook 编码；按 Ook 与 Brainfuck 的对应表执行后，输出只含大写字母和数字，符合 Base32 字符集。

接下来依次做 Base32 和 Base64 解码：

```python
import base64

# ook_output 是 Ook 解释器产生的 ASCII 字符串。
stage1 = base64.b32decode(ook_output)
png = base64.b64decode(stage1)

assert png.startswith(bytes.fromhex("89504e470d0a1a0a"))
with open("decoded.png", "wb") as f:
    f.write(png)
```

PNG 文件是可直接扫描的二维码：

![依次解开 Ook、Base32 与 Base64 后得到的最终二维码](./HGAME2020-签到题ProPlus-wp/decoded-qr.png)

扫码得到：

```text
hgame{3Nc0dInG_@lL_iN_0Ne!}
```

## 方法总结

- 核心技巧：严格按提示拆分多层链条，并在每一层用输出字符集或文件头验证下一种编码。
- 识别信号：`data:text;ook` 指向 Ook，Base32 只含大写字母和数字，Base64 解码后出现 PNG 文件签名。
- 复用要点：每解一层就保存中间结果并检查类型；不要把长编码截图继续保留在 WP 中，可复制的文本和最终二维码更有价值。
