# Robots

## 题目简述

站点根路径重定向到 `/robots`，页面文字继续询问 robots 在哪里。虽然仓库没有官方 WP，Flask 源码明确给出真正的 flag 路由。

## 解题过程

访问标准爬虫规则路径：

```text
/robots.txt
```

源码中该路由直接返回 `flag.txt`：

```python
@app.route('/robots.txt')
def flag():
    return open('flag.txt').read()
```

响应为：

```text
n00bz{1_f0und_7h3_r0b0ts!}
```

## 方法总结

题名和页面文案都在提示 Web 约定路径。`robots.txt` 本来就是公开资源，不能用于存放秘密；它适合提供爬虫指令，不是访问控制机制。
