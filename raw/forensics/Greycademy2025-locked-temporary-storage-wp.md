# Locked Temporary Storage

## 题目简述

题目提供域控的 `ntds.dit` 及 ESE 日志，并另外给出一份 BitLocker 加密的公司电脑磁盘。关键线索是域加入设备可以把 BitLocker 恢复信息作为计算机对象的子对象保存在 Active Directory 中；先从目录数据库取出 48 位恢复口令，再解锁磁盘寻找临时目录里的 flag。

## 解题过程

使用 DIT Explorer 以只读方式打开 `dist/NTDS/ntds.dit`。本题数据库可以直接打开，不需要破坏性修复；若副本处于 dirty shutdown 状态，才应先在副本上结合随附日志恢复。

在域树的 `Computers/GREYPUTER` 下找到类为 `ms-FVE-RecoveryInformation` 的子对象：

```text
CN=2026-02-07T04:02:18+08:00{1439E6B9-C00D-4795-B7BE-A0B4B8B7BD68},
CN=GREYPUTER,CN=Computers,DC=forest,DC=grey
```

查看该对象的 `msFVE-RecoveryPassword` 属性，得到：

```text
271942-019415-647427-146025-040634-614416-364540-500192
```

在 Windows 中可把镜像挂载为卷后执行：

```powershell
manage-bde -unlock X: -RecoveryPassword 271942-019415-647427-146025-040634-614416-364540-500192
```

也可在取证工具中把同一口令作为 BitLocker recovery password 提交。官方解题记录指出，解密卷根目录有提示，目标文件位于：

```text
C:\Users\GreyCat\AppData\Local\Temp\FLAG\flag.txt
```

文件内容为：

```text
grey{bitl0cker_keys_can_b3_stored_in_AD!}
```

仓库没有内置磁盘镜像，当前外置短链只返回已经过期的签名下载地址；因此本地复现完整验证到“从原始 `ntds.dit` 精确提取恢复口令”，磁盘内路径和最终文本则由仓库官方解题说明与题目 flag 交叉核对，没有伪称重新挂载了缺失镜像。

## 方法总结

BitLocker 取证不应只在被加密主机本身寻找密钥。域环境中的 `ms-FVE-RecoveryInformation` 子对象会保存恢复 GUID、卷 GUID 和 `msFVE-RecoveryPassword` 等属性。处理离线 `ntds.dit` 时优先保留原件、只读打开，并把数据库一致性恢复限制在副本上。
