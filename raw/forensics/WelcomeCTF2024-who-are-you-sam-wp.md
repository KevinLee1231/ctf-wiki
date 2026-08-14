# Who are you??? SAM!

## 题目简述

附件是 Windows SAM 注册表配置单元。题目要求确定管理员账户的姓名与创建时间，并按 `firstname_lastname_hhmmss` 组成 flag。

## 解题过程

使用 RegRipper 解析 SAM，可把二进制注册表结构转换为账户、RID、SID 和时间字段等可读记录：

```bash
rip.pl -r SAM -p samparse > sam.txt
```

先在解析结果中定位本地 Administrators 组，再查看其成员 SID/RID。不要把内置 `Administrator` 名称直接当作答案；应沿管理员组的成员标识回查实际用户记录。对应账户为 `James Rockya`，其账户创建时间为：

```text
11:56:48
```

按题目要求转为小写、用下划线连接并删除时间中的冒号：

```text
grey{james_rockya_115648}
```

## 方法总结

SAM 题的关键是把“组成员关系”和“用户详细记录”关联起来：先确认哪个 SID 具有管理员权限，再从该 SID 的用户记录取姓名与时间。只浏览用户名列表无法证明管理员身份。
