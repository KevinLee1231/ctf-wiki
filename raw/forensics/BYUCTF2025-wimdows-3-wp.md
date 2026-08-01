# Wimdows 3

## 题目简述

题目沿用 [Wimdows 虚拟机证据](https://byu.box.com/v/byuctf-wimdows)，要求确定攻击者新建账号后把它加入了哪个本地组。事件和攻击复现记录都表明新增账号名为 `phasma`。

## 解题过程

在 Windows Security/Event Log 中围绕 `phasma` 搜索账号创建与本地组成员变更事件。对应记录显示攻击者执行的等价命令为：

```powershell
net user phasma f1rst0rd3r! /add
net localgroup "Remote Desktop Users" phasma /add
```

同时还修改了 `Winlogon\SpecialAccounts\UserList` 隐藏该账号，但这只影响登录界面可见性，不改变组成员结论。组名为：

```text
Remote Desktop Users
```

按题目格式得到：

```text
byuctf{Remote Desktop Users}
```

## 方法总结

- 核心技巧：把账号创建、组成员修改和注册表隐藏操作按时间关联，确定新账号的远程登录权限。
- 识别信号：新建本地用户后紧接着出现本地组变更，是持久化或远程访问准备的常见链条。
- 复用要点：答案应来自组变更事件的目标组字段；账号被隐藏不等于被禁用，也不改变其权限。
