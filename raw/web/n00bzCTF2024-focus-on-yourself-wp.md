# Focus-on-yourSELF

## 题目简述

仓库没有官方 WP。图片查看接口把查询参数直接拼进 `/srv/uploads/{image}` 后打开，未做规范化或目录边界检查；题名中的 `SELF` 提示读取当前进程的 `/proc/self/environ`。

## 解题过程

访问：

```text
/view?image=../../proc/self/environ
```

拼接路径为 `/srv/uploads/../../proc/self/environ`，规范化后就是 `/proc/self/environ`。服务把读取到的字节 Base64 编码，并放入 HTML：

```html
<img src="data:image/jpeg;base64, ...">
```

环境文件并非 JPEG，浏览器可能只显示损坏图片；应查看响应源代码，提取逗号后的 Base64 并解码。NUL 分隔的环境变量中包含 `FLAG=`。仓库只保留了 `REPLACETHIS` 占位值，因此我又对照比赛期间的公开解题页面截图逐字观察，确认实际比赛 flag 完整为：

```text
n00bz{Th3_3nv1r0nm3nt_det3rmine5_4h3_S3lF_13ab553151d3}
```

## 方法总结

检查上传文件名扩展名不能防止查看接口的路径穿越。应对用户路径做规范化后验证其父目录，或只接受服务端生成的 opaque ID；Base64 数据 URL 也不等于内容已被安全验证。
