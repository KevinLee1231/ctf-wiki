# BYUCTF 2022 - Fetaverse

## 题目简述

网站表面只有一页羊乳酪图片。题面夸赞其“设计”，提示从静态站点结构而不是图片隐写入手。

## 解题过程

在新标签页打开任一 `/img/fetaN.jpg`，再访问其父路径 `/img/`。Nginx 启用了 `autoindex on`，目录清单除九张正常图片外还暴露未在页面链接的目录：

```text
memes/
```

进入 `/img/memes/` 后依次查看文件。前六张是普通奶酪梗图，第七张 `meme7.png` 是一张 Cheese Man 卡牌，底部直接写有 flag：

![目录遍历后在 meme7 中找到的 Cheese Man flag](./BYUCTF2022_Fetaverse/final-cheese-meme.png)

```text
byuctf{welc0me_t0_the_fetaverse}
```

无需对九张 feta 图片做 LSB、EXIF 或压缩包检查；关键泄露来自 Web 服务器目录索引。

## 方法总结

静态资源 URL 本身会暴露目录层级。发现可列目录时，应检查未被 HTML 引用的子目录、备份文件和顺序异常资源；本题最后一张图片只是明文载荷，目录枚举才是核心障碍。
