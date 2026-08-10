# ezWord

## 题目简述

题目附件是 Word 文档，表面内容不是终点。DOCX 本质上是 ZIP 容器，其内部藏有两张几乎无法肉眼区分的图片和一个加密压缩包。两图差分中嵌入了盲水印密码；解压后还需依次解码 Spam Mimic 伪装文本和 Unicode 位移密文，才能恢复 flag。

## 解题过程

### 将 DOCX 当作 ZIP 检查

复制附件并将扩展名改为 `.zip`，或直接用支持 ZIP 的解压工具打开。在容器中可找到：

```text
100191209_p0.jpg
image1.png
secret.zip
恭喜.txt
```

`恭喜.txt` 提示，需要根据两张图片解出新压缩包的密码，解压后再对文本做两层解码。

PDF 中展示的原图和嵌入盲水印后的图如下。两者的差异不能靠普通肉眼稳定识别，这正是盲水印的隐蔽性所在。

![作为盲水印基准的初音未原图](HGAME2024-ezWord-wp/blind-watermark-original.jpeg)

![外观与原图几乎一致的盲水印载体图](HGAME2024-ezWord-wp/blind-watermark-stego.jpeg)

### 从两张图片提取盲水印

官方题解使用 [BlindWaterMark](https://github.com/chishaxie/BlindWaterMark) 的 Python 3 脚本。该工具的核心用法是同时提供原图、含水印图和输出路径，通过两图频域差异恢复水印：

```bash
python bwmforpy3.py decode \
  100191209_p0.jpg \
  image1.png \
  wm_from_hui.png
```

提取结果是一张文字水印图，内容为：

```text
T1hi3sI4sKey
```

这个字符串长度为 11，同时包含大写字母、小写字母和数字，与压缩包提示完全一致。用它解开 `secret.zip`。

### 解码 Spam Mimic 伪装文本

解压后得到一大段看似垃圾邮件的英文，开头类似：

```text
Dear E-Commerce professional ; This letter was specially
selected to be sent to you . We will comply with all
removal requests ! This mail is being sent in compliance
with Senate bill 1620 ; Title 3 ; Section 308 !
```

这不是用来阅读的自然语言，而是 Spam Mimic 以词句选择和标点变化承载比特的伪装文本。将整段文本送入 Spam Mimic 的 decode 功能，恢复出：

```text
籱籰籪籶籮粴簹籴籨粂籸籾籨籼簹籵籿籮籨籪籵簺籨籽籱簼籨籼籮籬类簼籽粆
```

一大段 Spam Mimic 输出只是这层编码的载体，具体英文句子不影响后续步骤，因此不在本文重复粘贴全部 158 行伪装邮件。

### 爆破 Unicode 位移

上一层输出是等长 Unicode 位移密文。官方出题过程将这层称为 `ROT 8000`，但对实际字符串求解时，可直接在 Unicode 码点空间枚举逆向位移，并用已知 flag 前缀作为判据：

```python
ciphertext = (
    "籱籰籪籶籮粴簹籴籨粂籸籾籨籼簹籵籿籮"
    "籨籪籵簺籨籽籱簼籨籼籮籬类簼籽粆"
)


def unicode_shift(text, shift):
    return "".join(
        chr((ord(character) + shift) % 0x110000)
        for character in text
    )


for shift in range(-65535, 65536):
    candidate = unicode_shift(ciphertext, shift)
    if candidate.startswith("hgame{"):
        print(f"shift: {shift}")
        print(candidate)
        break
```

输出：

```text
shift: -31753
hgame{0k_you_s0lve_al1_th3_secr3t}
```

## 方法总结

- 处理 Office Open XML 附件时，先检查 ZIP 容器、`word/media` 与额外嵌入文件，不要只看 Word 界面显示的文本。
- 双图盲水印需要原图和载体图成对输入；图像在肉眼下几乎相同并不表示像素或频域系数完全相同。
- Spam Mimic 的垃圾邮件是编码容器，解码必须使用完整文本，不能随意删掉标点、换行或句子。
- 最后一层虽然显示为中文乱码，但长度与码点差分都表明它是可逆的恒定 Unicode 位移；已知 flag 前缀可用于快速验证候选。
