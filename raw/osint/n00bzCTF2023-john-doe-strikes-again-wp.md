# John Doe Strikes Again

## 题目简述

题目给出重复异或密钥、密文和账号标识。解密出的对话会引导选手依次关联 Spotify、Discord、比赛平台和 Wayback Machine，最终在历史页面中取得另一段密文。

## 解题过程

先让密钥 `YouCanNeverCatchJohnDoe!` 循环覆盖密文：

```python
plain = bytes(
    value ^ key[index % len(key)]
    for index, value in enumerate(ciphertext)
)
print(plain.decode())
```

明文提到 John Doe 喜欢音乐。搜索题面给出的账号标识 `31zdugxvkayexc4hzqhixxcfxb4y`，可定位到 Spotify 账号；播放列表名称与密钥相同，说明账号关联正确。列表描述也是重复密钥异或密文，解密后提示关注头像。

头像是 Discord 标志，于是在比赛 Discord 中定位 John Doe。其个人简介要求查看比赛平台资料；平台上的战队名仍是密文，解密得到 `Think Way Back`。据此查询 Wayback Machine 中该资料页的历史快照，找到曾经挂出的个人网站。网站内最后一段密文使用相同密钥解密，得到：

```text
n00bz{n0_0n3_c4n_3sc4p3_MR.051N7,_n0t_3v3n_J0HN_D03!}
```

## 方法总结

这是一条跨平台账号关联链。每一步都应利用独立特征验证身份连续性，例如相同播放列表名、头像提示和可解密的战队名；重复异或只是贯穿整条调查链的解码工具。
