# BYUCTF 2023 - Legoclones 1

## 题目简述

已知网名 `Legoclones` 使用超过十年，并曾“收养”一个以《星球大战》设定为主题、经营约三年的大型网站。要求找出该网站 URL。

## 解题过程

直接搜索带空格的 `lego clones` 会混入大量积木商品；使用精确网名 `"legoclones"`，并加入 `Star Wars`、`wiki`，可以定位 [Clone Trooper Wiki](https://clonetrooper.fandom.com/)。旧 Wikia 域名经历过迁移，因此比赛接受三种等价地址：

```text
byuctf{https://clone.wikia.com}
byuctf{https://clonetrooper.wikia.com}
byuctf{https://clonetrooper.fandom.com}
```

当前最稳定的规范地址是第三个。

## 方法总结

用户名 OSINT 要保留精确拼写并结合主题词缩小结果。历史平台迁移会产生多个等价域名，写 WP 时应说明哪些是旧域、哪些是当前域，而不是把重定向误认为不同站点。
