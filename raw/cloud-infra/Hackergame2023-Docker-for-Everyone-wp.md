# Docker for Everyone

## 题目简述

环境把低权限用户 `hg` 加入了宿主的 `docker` 组，并提供本地 Alpine 镜像。flag 位于宿主 `/dev/shm/flag`，根目录的 `/flag` 是指向它的软链接。目标是说明能直接访问 Docker daemon 的用户实际上拥有近似宿主 root 的能力。

题目外层用 QEMU 启动虚拟机，并把 flag 文件作为可写 virtio 块设备附加；虚拟机内的 Docker daemon 以 root 权限创建容器。决定性障碍是 Docker 控制面、宿主挂载和 IPC 命名空间的权限边界，因此归入 `cloud-infra`。

## 解题过程

Docker daemon 由 root 运行，客户端提交的容器配置可以要求 daemon 把任意宿主路径 bind mount 进容器。直接把宿主根目录挂到 `/outside`：

```bash
docker run --rm -it -v /:/outside alpine
```

进入容器后，`/outside` 对应宿主 `/`。但是直接读取 `/outside/flag` 可能失败：该路径是绝对软链接 `/dev/shm/flag`，在容器中解析时会落到容器自己的 `/dev/shm`，而不是宿主挂载树中的 `/outside/dev/shm`。可以绕过软链接直接读真实文件：

```bash
cat /outside/dev/shm/flag
```

另一种方式是在创建容器时共享宿主 IPC 命名空间：

```bash
docker run --rm -it --ipc=host -v /:/outside alpine
cat /outside/flag
```

此时容器的 `/dev/shm` 与宿主相同，`/outside/flag -> /dev/shm/flag` 能正确解析到目标文件。

这里不需要容器逃逸漏洞：bind mount 本身就是 Docker daemon 受授权执行的宿主 root 操作。只要低权限用户能控制 daemon，就能挂载宿主文件系统、修改宿主配置或启动更高权限的容器。读取到动态 flag 即证明权限边界已被跨越。

## 方法总结

- 核心技巧：借 root Docker daemon 将宿主 `/` bind mount 到攻击者控制的容器中，再处理跨命名空间的绝对软链接。
- 识别信号：低权限账号属于 `docker` 组或能访问 `/var/run/docker.sock`，且可自由指定镜像、挂载和 namespace 参数。
- 复用要点：`docker` 组不是普通资源访问组，通常等价于宿主 root。多用户环境应使用 rootless 容器、受控编排接口或严格隔离的虚拟机，而不能把 Docker socket 直接下放给不可信用户。
