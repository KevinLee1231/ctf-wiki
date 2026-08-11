# Bad Policies

## 题目简述

压缩包中是从 Windows 域环境导出的组策略首选项（Group Policy Preferences, GPP）文件。旧式 GPP 会把可逆的 `cpassword` 写入 XML；找到该字段并用已知的 GPP 解密算法恢复明文即可。

## 解题过程

解压后在策略目录中搜索 `cpassword`，例如：

```text
grep -r "cpass" .
```

得到的 XML 属性值为：

```text
B+iL/dnbBHSlVf66R8HOuAiGHAtFOVLZwXu0FYf+jQ6553UUgGNwSZucgdz98klzBuFqKtTpO1bRZIsrF8b4Hu5n6KccA7SBWlbLBWnLXAkPquHFwdC70HXBcRlz38q2
```

该条目位于 `Machine/Preferences/Groups/Groups.xml`，目标账号名为 `Backup`；这说明它是域策略中下发的本地组/用户偏好，而非普通注册表值。

将该值交给 GPP 专用解密工具：

```text
gpp-decrypt B+iL/dnbBHSlVf66R8HOuAiGHAtFOVLZwXu0FYf+jQ6553UUgGNwSZucgdz98klzBuFqKtTpO1bRZIsrF8b4Hu5n6KccA7SBWlbLBWnLXAkPquHFwdC70HXBcRlz38q2
```

明文是题目所需凭据，最终 flag 为：

```text
DUCTF{D0n7_Us3_P4s5w0rds_1n_Gr0up_P0l1cy}
```

## 方法总结

GPP 的 `cpassword` 不是不可逆口令哈希：历史实现使用公开密钥，任何拿到策略文件的人都能还原它。取证时应把 `Groups.xml`、`Services.xml`、`ScheduledTasks.xml` 等策略偏好文件列为凭据排查对象；防守上应删除遗留 `cpassword` 并轮换相关账号密码。
