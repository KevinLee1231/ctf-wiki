# BYUCTF 2022 - Black

## 题目简述

站点由 Vue/Nuxt 构建，页面在若干 hover 区域中显示超宽的二维码切片。拼图只负责进入隐藏路由；隐藏路由又加载一段被 CSS 缩到 3×3 像素的视频，最终答案来自视频中的七个单词。

## 解题过程

在开发者工具中检查 hover 元素和 `_nuxt/img/` 资源，可找到 `row-1-column-1` 到 `row-8-column-1` 八张 2999×375 图片。按编号从上到下拼接，得到完整的圆角样式二维码：

![按 row 1 至 row 8 垂直拼接出的圆角二维码](./BYUCTF2022_Black/assembled-rounded-qr.png)

实际扫码得到 `https://qrco.de/bcwQZu`。该短链目前显示账号已停用，但赛事本地 Nuxt 文件提供了独立证据：路由表含 `/index_`，对应 HTML 引用 `videos/loadButton.85316c6.mp4`，CSS 明确设置：

```css
video { height: 3px !important; width: 3px !important; }
```

视频因体积未提交 Git，官方保留了 [Box 附件](https://app.box.com/s/rbdoh3eyhcli15pkavvtvjzidaol8t4t)，下载后应放到 `_nuxt/videos/`。完整观看可找到七个散落单词，将其按加拿大国歌歌词顺序排列为：

```text
Oh Canada Our Home And Native Land
```

最终提交：

```text
byuctf{Oh Canada Our Home And Native Land}
```

## 方法总结

本题是两级前端资源发现：hover 区域泄露二维码切片，隐藏路由再用 CSS 隐藏视频。短链失效后，路由 bundle、HTML 和 CSS 仍能证明入口机制；缺失视频中的七词及顺序则由官方记录完整保留。
