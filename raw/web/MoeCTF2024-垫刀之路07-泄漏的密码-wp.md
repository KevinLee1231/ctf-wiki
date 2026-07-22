# 垫刀之路07：泄漏的密码

## 题目简述

Flask 应用以 `debug=True` 启动，同时把 Werkzeug 调试器 PIN 直接显示在首页。拿到 PIN 后可进入 `/console` 执行 Python。部署脚本还揭示了一个容易误判的细节：真实 flag 被写入 `/app/flag` 后，环境变量 `FLAG` 被改成诱饵，因此必须读文件而不是继续查环境变量。

## 解题过程

源码中的启动方式为：

```python
app.run(debug=True, host="0.0.0.0", port=80)
```

首页模板接收并显示 `get_pin()` 的结果。访问 `/console`，输入页面泄漏的 PIN 解锁调试控制台。控制台没有额外沙箱，可以直接执行 Python：

```python
import os
os.popen("cat /app/flag").read()
```

这里使用 `os.popen(...).read()` 是为了把命令标准输出作为表达式结果显示；`os.system()` 通常只返回退出状态，在网页控制台里可能看不到 flag 文本。

题目部署脚本的顺序是：

```bash
sed -i "s/moectf{test}/$FLAG/g" /app/flag
export FLAG=fake_flag
FLAG=fake_flag
```

所以 `/flag` 是诱饵，当前环境变量也是 `fake_flag`，真实内容位于工作目录的 `/app/flag`。

## 方法总结

调试模式暴露的风险不只是报错堆栈；若 PIN 同时泄漏，Werkzeug Console 就是直接的代码执行入口。获得执行能力后仍要核对 Dockerfile、entrypoint 和启动脚本，因为 flag 可能在启动阶段被落盘后再用假值覆盖环境变量。本题的完整链为“PIN 泄漏 → `/console` → Python 执行 → 读取 `/app/flag`”。
