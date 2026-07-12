# real_cloud_serverless

## 题目简述

本题是 `real_cloud_storage` 的后续云原生利用链。第一步拿到 hint `only cluster admin could see the next flag` 后，目标从对象存储 SDK 漏洞转向 Kubernetes/serverless 环境。通过前一题 XXE 读取 service account 信息，可确认当前命名空间为 `fission-function`，后端运行在 Fission serverless 框架中。

核心利用链是：用 XXE/SSRF 访问未鉴权的 `fission-controller`，读取并更新 Fission `Environment` 资源；通过 `podspec.securityContext.privileged=true` 创建特权函数容器逃逸到 node；再读取拥有 `cluster-admin` 绑定的 `fission-controller` ServiceAccount token，最终访问集群 secrets 中的 flag。

## 解题过程

题目源码见 [`Li4n0/My-CTF-Challenges/D^3CTF2021_real_cloud`](https://github.com/Li4n0/My-CTF-Challenges/tree/master/D%5E3CTF2021_real_cloud)。下文保留了源码和官方 WP 中的 serverless 利用主线：XXE 读取 ServiceAccount、访问 Fission controller、修改 Environment PodSpec、进入 cluster-admin。

拿到第一步 flag 的同时会得到 hint：`only cluster admin could see the next flag`。结合后端部署在 serverless 环境中，可以判断后续目标是从函数容器进入 Kubernetes 集群控制面，并取得 `cluster-admin` 权限。

对 Kubernetes 环境做信息收集时，优先通过上一题的 XXE 读取 `/var/run/secrets/kubernetes.io/serviceaccount/` 下的文件。读取 `namespace` 后可确认当前命名空间为 `fission-function`。[`Fission`](https://fission.io/) 是基于 Kubernetes 的开源 serverless 框架，函数运行、环境配置和调度都由集群内控制组件完成，因此接下来应围绕 Fission 控制面找可利用点。

Fission 的基础使用模式是先创建 `Environment`，再在该环境上创建函数：

```bash
# 使用 fission/python-env 镜像创建名为 python 的 environment
fission env create --name python --image fission/python-env

# 创建一个名为 hello_world、代码为 code.py 的函数
fission fn create --name hello_world --env python --code code.py
```

根据 Fission 的 [controller 组件文档](https://docs.fission.io/docs/concepts/components/core/controller/)，客户端请求会发送到 `fission-controller`，再由它通过 Kubernetes API 创建或更新函数相关资源：

![Fission controller 调度架构](<./D3CTF2021-real-cloud-serverless-wp/fission_controller_flow.png>)

实际测试可发现 `fission-function` 容器到 `fission-controller` 之间默认没有网络隔离，按 Kubernetes 的 `service.namespace` 访问方式请求 `controller.fission` 即可到达控制器。更严重的是 controller 默认未鉴权；如果能让函数容器发请求，就可以通过 controller 的 REST API 控制 Fission 资源。Fission controller API 的入口可从源码中的 [`pkg/controller/api.go`](https://github.com/fission/fission/blob/master/pkg/controller/api.go) 对照确认。

此时手里有两个互补能力：

1. NOS Client 参数可控造成的 SSRF：`PUT` 请求，path 和 body 可控，但没有回显。
2. XXE 带来的 SSRF：可发起 `GET` 请求，并能通过外带方式拿到响应 body。

Fission controller 的更新接口需要 JSON body。虽然 NOS SDK 发出的 `Content-Type` 是 `application/octet-stream`，但 controller 实现中直接对请求 body 做 JSON 解码，没有严格校验 `Content-Type`，因此仍可用 `PUT` SSRF 更新资源。更新资源还需要目标资源名、`uid` 和 `generation` 等字段，这些可先通过 XXE 访问 `controller.fission/v2/environments` 获取。

下一步是修改 `Environment`。根据 Fission 的 [PodSpec 文档](https://docs.fission.io/docs/spec/podspec/)，`Environment.spec.runtime.podspec` 可以影响函数容器的 `securityContext`。将容器改为特权模式后即可进一步做容器逃逸：

```json
{
  "spec": {
    "version": 2,
    "runtime": {
      "image": "fission/jvm-env",
      "podspec": {
        "containers": [
          {
            "name": "hack",
            "image": "fission/jvm-env",
            "command": ["/bin/sh", "-c", "<escape-and-reverse-shell>"],
            "securityContext": {
              "privileged": true
            }
          }
        ]
      }
    }
  }
}
```

完整利用流程是：先通过 XXE 读取 `controller.fission/v2/environments`，选择一个已有 `Environment` 并保留其 metadata；再把 `spec.runtime.podspec.containers[].securityContext.privileged` 改成 `true`，命令字段换成逃逸脚本或反弹 shell；最后将修改后的 JSON 通过 `PUT` SSRF 发送到 `controller.fission/v2/environments/<env-name>`，等待特权函数容器启动并控制宿主机 node。

拿到 node 权限后还需要升级到 `cluster-admin`。Fission 默认部署中 `fission-controller` 使用的 ServiceAccount 是 `fission-svc`，并绑定了 `cluster-admin`：

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fission-crd
subjects:
  - kind: ServiceAccount
    name: fission-svc
    namespace: fission
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

因此在宿主机上读取 `fission-controller` 容器内的 `/var/run/secrets/kubernetes.io/serviceaccount/token`，即可用该 token 以 `cluster-admin` 身份访问 Kubernetes API。最终 flag 位于集群 secrets 中。

## 方法总结

- 核心技巧：把 SSRF/XXE 作为 Kubernetes 内网访问能力，利用 Fission controller 默认未鉴权和高权限 ServiceAccount 完成 serverless 到 cluster-admin 的升级。
- 识别信号：函数容器内存在 `/var/run/secrets/kubernetes.io/serviceaccount`，命名空间包含 `fission`，controller 服务可跨命名空间访问且 REST API 未鉴权。
- 复用要点：云原生题要同时检查网络隔离、控制面鉴权、PodSpec 可控字段和 RBAC 最小权限。能控制 serverless 环境资源时，优先寻找 `securityContext`、镜像、命令和 ServiceAccount 相关字段。

