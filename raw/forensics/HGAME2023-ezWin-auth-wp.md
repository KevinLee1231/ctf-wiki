# ezWin auth

## 题目简述

本题继续分析 `win10_22h2_19045.2486.vmem`。内存中的命令行提示第二段 flag 是“当前用户的 NT hash”，因此要先确定当时登录或操作记事本的用户，再从注册表凭据材料中提取对应的 NTLM 哈希。

## 解题过程

先枚举进程命令行并搜索 flag 线索：

```sh
python3 vol.py \
  -f win10_22h2_19045.2486.vmem \
  windows.cmdline.CmdLine | grep -i flag
```

内存中残留的命令行给出提示：

```text
flag2 is nthash of current user.txt
```

接着查看登录会话，并用 `notepad` 进程缩小范围：

```sh
python3 vol.py \
  -f win10_22h2_19045.2486.vmem \
  windows.sessions.Sessions | grep -i notepad
```

对应用户名为 `Noname`。最后运行哈希提取插件：

```sh
python3 vol.py \
  -f win10_22h2_19045.2486.vmem \
  windows.hashdump.Hashdump
```

关键记录为：

```text
Noname  1000  aad3b435b51404eeaad3b435b51404ee  84b0d9c9f830238933e7131d60ac6436
```

前一个固定值是空 LM hash，最后一列才是该用户的 NT hash。按题意直接包入 flag 格式：

```text
hgame{84b0d9c9f830238933e7131d60ac6436}
```

## 方法总结

`hashdump` 的输出可能包含多个账户，不能看到哈希就随便提交。应先从会话、进程所有者或交互痕迹确定“当前用户”，再匹配对应 RID 与 NT hash。也要区分 LM 和 NT 两列：`aad3b435...` 常表示未存储 LM 密码，并不是本题答案。
