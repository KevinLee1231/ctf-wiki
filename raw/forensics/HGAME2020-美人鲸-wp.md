# 美 人 鲸

## 题目简述

题目给出一个 Docker 镜像，flag 被拆成多层文件系统线索：环境变量把目标指向压缩包，压缩包内的说明要求检查 shell 历史，历史又提示在 `/etc` 搜索 ZIP 密码，最终从 SQLite 数据库中读取 flag。决定性工作是恢复并关联容器镜像中的证据，因此归入 Forensics。

## 解题过程

题目镜像名为：

```text
zhouweitong/hgame2020-misc:week3
```

历史环境中可拉取并启动容器：

```bash
docker pull zhouweitong/hgame2020-misc:week3
docker run --rm -it -p 8000:80 zhouweitong/hgame2020-misc:week3 /bin/bash
```

容器页面提示 flag 与环境变量有关，但直接执行 `echo "$FLAG"` 得到的不是 flag，而是“寻找 `flag.tar.gz`”的下一层线索。搜索整个文件系统：

```bash
find / -name 'flag.tar.gz' 2>/dev/null
```

文件位于：

```text
/usr/share/man/man8/flag.tar.gz
```

解包后得到 `flag.zip` 和 `README`。README 的关键信息是查看 shell 历史，因此继续检查：

```bash
history
```

历史记录表明 ZIP 密码被放在 `/etc` 的某个文本中。定向搜索即可定位：

```bash
grep -Rni 'Zip Password' /etc 2>/dev/null
```

命中 `/etc/issue`：

```text
Zip Password: cfuzQ3Gd6gqKG@$N
```

用该密码解开 `flag.zip`，得到 `flag.db`。这是 SQLite 数据库，可先列出表，再读取 `hgame2020` 表：

```bash
sqlite3 flag.db '.tables'
sqlite3 flag.db 'SELECT * FROM hgame2020;'
```

查询结果为：

```text
hgame{v3RWI3qSpcKZhp^xv$kaBhNjVqxk##3e}
```

## 方法总结

- 核心链路：容器环境变量、镜像文件系统、shell 历史、系统标识文件、加密压缩包与 SQLite 数据库逐层关联。
- 关键方法：每一层说明文件都应转化为可验证的下一步搜索条件，不能只依赖手工浏览目录。
- 分类依据：Docker 只是证据载体，解题并未利用云控制面或容器逃逸；核心是从已取得的镜像中恢复事实，因此属于 Forensics。
