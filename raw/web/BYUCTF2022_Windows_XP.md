# BYUCTF 2022 - Windows XP

## 题目简述

题目是一个会随窗口尺寸变化的网页。页面请求 `/images/<image>` 获取二维码；客户端线索和响应式行为暗示应检查图片接口的查询参数，而不是只扫描默认二维码。

## 解题过程

缩小浏览器窗口或阅读 `/js/challenge.js`，可恢复提示短语：

```text
steve is so cool
```

服务端 `main.py` 的图片路由直接比较查询参数：

```python
@app.route('/images/<image>')
def get_image(image):
    if request.args.get('flag') == 'steve is so cool':
        return send_file('images/flag.png')
    return send_file('images/normal.png')
```

因此访问：

```text
/images/default?flag=steve+is+so+cool
```

服务器返回另一张二维码：

![带正确查询参数后接口返回的 flag 二维码](./BYUCTF2022_Windows_XP/flag-response-qr.png)

扫码得到：

```text
byuctf{size_matters_not_look_at_me}
```

## 方法总结

决定性机制是客户端可见秘密控制服务端资源分支。响应式布局只是隐藏提示的方式；直接审查 JavaScript 和 Flask 路由能确认精确参数名、空格编码以及成功响应文件。
