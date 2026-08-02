# TJCTF2022 photoable

## 题目简述

图片下载路由为 `/image/:imageid/download`。服务端把 Express 已解码的 `imageid` 直接拼进 `photobucket/${getFileName(imageid)}`，再交给 `path.join` 和 `res.sendFile`。编码后的斜杠可进入单个路由参数，解码后则变成 `../`，造成目录穿越。

## 解题过程

使用 `..%2Fflag.txt` 作为 `imageid`。路由匹配阶段 `%2F` 尚未作为路径分隔符参与匹配，因此请求仍落入下载处理器；参数解码后，`imageid` 变成 `../flag.txt`。该值不在 `uuid2ext` 中，`getFileName` 不追加扩展名，于是：

```javascript
path.join(__dirname, 'photobucket/../flag.txt')
```

解析到服务器目录中的 `flag.txt`。请求如下：

```bash
curl 'http://target/image/..%2Fflag.txt/download'
```

`sendFile` 返回文件内容 `tjctf{1fram3_1fl4g}`。

## 方法总结

路由参数在“匹配时的编码形式”和“业务使用时的解码形式”之间存在安全边界，编码斜杠经常用于跨过这一边界。文件下载接口应先确认 ID 符合 UUID 白名单，再从服务器维护的映射中取路径；若必须拼路径，则规范化后验证目标仍在固定根目录下。
