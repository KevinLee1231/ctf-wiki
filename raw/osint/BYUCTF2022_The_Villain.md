# BYUCTF 2022 - The Villain

## 题目简述

附件包含一张文档截图和加密 ZIP。目标是从截图元数据找到在线文档，再由隐藏姓名关联社交账号、恢复压缩包口令。

## 解题过程

先运行：

```bash
exiftool 'Screenshot 2022-05-10 201759.png'
```

元数据 `Comment` 中保存了题目所示 [Google Doc](https://docs.google.com/document/d/1BnPoqSl7GQOST__5iR7Xv7lYB8kSIxFsLYSq9PXaJFY/edit) 的 URL。打开文档并全选文本，可显出白色字体隐藏的姓名：

```text
Astoria Villin
```

搜索该姓名可定位 Twitter 账号 `@AstoriaVillin`。账号生活线索给出 `HoppyBoi`，它正是本地 `encrypted.zip` 的口令。实际解压验证：

```bash
7z x -pHoppyBoi encrypted.zip
```

压缩包内只有 `flag.png`，手写内容为：

```text
byuctf{b3t_Y0U_c4nt_f1nd_D33ZNU7S!}
```

## 方法总结

完整链条是“图片元数据 URL → 文档白字姓名 → 公开账号生活线索 → ZIP 口令”。外部页面的重要内容已经写入正文；最终压缩包仍在仓库中，并已用推导出的口令实测解包。
