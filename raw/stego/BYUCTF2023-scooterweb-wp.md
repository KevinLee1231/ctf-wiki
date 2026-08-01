# BYUCTF 2023 - ScooterWeb

## 题目简述

网页轮播包含八张 PNG。每张图的 `iTXt/zTXt` 注释中藏有一段 192 个十六进制字符的消息；需要组合其中七段，排除一段干扰数据。

## 解题过程

图片本身只是载体，视觉内容不参与解码，因此不保留八张车辆照片。先批量读取元数据：

```bash
exiftool -Comment *.png
# 或
identify -verbose *.png | grep -i 'comment:'
```

把八个十六进制串转成等长字节串。异或全部八段得到 `all_xor`，再依次与每一段异或；这等价于每次排除一个候选：

```python
all_xor = blocks[0]
for block in blocks[1:]:
    all_xor = bytes(a ^ b for a, b in zip(all_xor, block))

for omitted in blocks:
    candidate = bytes(a ^ b for a, b in zip(all_xor, omitted))
    try:
        print(candidate.decode())
    except UnicodeDecodeError:
        pass
```

排除第五张图片的注释后得到：

```text
byuctf{ThisAintNoKickOrMobilityScooter}
```

被排除的消息同时充当 ScooterAdmin 附件隐藏目录名，是跨题线索而非本题 flag 内容。

## 方法总结

多份等长随机状数据常见 XOR secret sharing。未知哪一份是干扰时，可以先异或全集，再逐一“异或回去”做 leave-one-out 测试，并以 UTF-8 可解码性、前缀和语义验证候选。
