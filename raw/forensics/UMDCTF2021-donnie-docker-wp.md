# Donnie Docker

## 题目简述

题目提供一台主机环境。当前用户虽然不是 root，却属于 `docker` 组；本机还存在名为 `secret_image` 的镜像。目标是利用 Docker 守护进程权限读取镜像或宿主机中的 flag。

## 解题过程

先确认当前用户组和镜像列表：

```bash
id
docker images
```

能访问 `/var/run/docker.sock` 的用户可以让守护进程启动高权限容器，实际权限通常等同于 root。对本题最直接的做法是运行已有镜像并读取其根文件系统：

```bash
docker run --rm secret_image cat /flag.txt
```

若镜像默认入口点阻止执行命令，可覆盖入口点：

```bash
docker run --rm --entrypoint /bin/sh secret_image -c 'cat /flag.txt'
```

文件内容为：

```text
UMDCTF-{h1dd3n_1m@g35}
```

也可以 `docker save secret_image` 后解包各层，在层 tar 中定位 `flag.txt`；这条路径不需要运行未知镜像。

## 方法总结

`docker` 组不是普通的低权限辅助组。能控制 Docker daemon，就能挂载宿主机目录、启动特权容器或读取任意镜像层。本题 flag 位于镜像的 `/flag.txt`，运行容器读取与离线解层都能形成可复现解法。
