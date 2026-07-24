# UMDCTF 2018 - Timed Trouble

## 题目简述

附件是 Windows 取证归档，仓库还保留了恶意 PowerShell、服务端代码和 flag 工件。题目重点是从计划任务导出中识别异常任务，并把任务动作与恶意脚本的下载、解码流程对应起来。

## 解题过程

解压归档并检查 `ScheduledTasks.csv`，最异常的记录为：

```text
Task:      \aoidsmfioamsdfoa\132940u12jp
Author:    joiejrwmadkfs
Account:   NETWORK SERVICE
Schedule:  every day at 00:00
Action:    powershell -encodedcommand ...
```

将动作中的 Base64 按 PowerShell `-EncodedCommand` 规则以 UTF-16LE 解码，得到的脚本 SHA-256 为：

```text
bbec4cb57e76820885a1c32f3fd0716ee609e3ce0c9550402a8694ca221ac837
```

它与仓库 `mal.ps1` 完全一致。脚本从历史地址 `167.99.224.34` 下载数据，解码函数 `dstr` 的步骤是：

```text
1. Base64 解码；
2. 第 0 字节作为尾部扩展长度；
3. key = 第 1 字节 XOR 0xaa；
4. 对第 i 个后续字节异或 (key + i) AND 0xff；
5. 跳过前两字节并按 Deflate 解压。
```

仓库 `flag.txt` 保存的明文为：

```text
UMDCTF-{power-on-a-schedule}
```

需要说明一个官方工件问题：根目录 `flag.bin` 与 `app/flag.bin` 长度不同，而且两者都不能按当前 `mal.ps1` 的 `dstr` 逻辑成功解压。因此本文能严格复现“计划任务 $\rightarrow$ 编码 PowerShell $\rightarrow$ 下载与解码算法”，最终明文则来自仓库同时发布的 `flag.txt`；不能声称现有 `flag.bin` 已被成功解密。

## 方法总结

计划任务取证应同时检查任务路径、作者、运行账户、触发器和动作。遇到源码与二进制工件不一致时，应把已验证链条和无法复现的部分明确分开；记录官方工件缺陷比强行拼出一个虚假的成功日志更可靠。
