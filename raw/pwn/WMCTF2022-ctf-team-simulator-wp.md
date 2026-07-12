# ctf-team_simulator

## 题目简述

题目给出一个由 `server.py` 包装的注册脚本解释器。服务端会读取选手输入的多行脚本，遇到 `END` 后把内容写入临时文件，再执行 `/home/ctf/pwn <临时文件>`。核心目标不是常规栈/堆利用，而是逆向这个由 flex 生成的“注册语言”，构造合法脚本触发 `admin load` 从指定路径读取 flag。

## 解题过程

本题是用 flex 编译生成的一种 “注册语言”。逆向时，会发现大量的 token 代号，从 1 到 17 分别为：

```c
#define     CREATE  1
#define     LOGIN   2
#define     USER    3
#define     TEAM    4
#define     LOGOUT  5
#define     JOIN    6
#define     SHOW    7
#define     NAME_SIGN   10
#define     PWD_SIGN    11
#define     EMAIL_SIGN  12
#define     PHONE_SIGN  13
#define     ADMIN_  14
#define     LOAD    15

#define     CONTENT 16
#define     HELP    17
```

对应的输入分别为

```c
"help"          {return(HELP);}
"exit"          {return(EXIT);}
"create"        {return(CREATE);}
"user"          {return(USER);}
"team"          {return(TEAM);}
"login"         {return(LOGIN);}
"join"          {return(JOIN);}
"show"          {return(SHOW);}
"admin"         {return(ADMIN_);}
"load"          {return(LOAD);}
"-p"            {return(PWD_SIGN);}
"-n"            {return(NAME_SIGN);}
"-e"            {return(EMAIL_SIGN);}
"-P"            {return(PHONE_SIGN);}
{content}       {return(CONTENT);}
```

附件中的 `server.py` 只负责收集脚本、执行二进制和清理临时文件，本身没有鉴权逻辑；真正的约束都在 `pwn` 二进制里。通过逆向可知，在使用 admin 时，可以从文件中读入 user，因此可以直接分析此 Load user 函数。不难发现，load user 的条件是队伍的成员大于 2 个，并且该用户的用户名必须为 `adm1n`。

因此直接按照指令格式进行队伍和用户的创建即可。输入的 exp 如下：

```
create user -n adm1n -p 123456 -P 1 -e nmsl
create user -n usr -p 654321 -P 2 -e wsnd
create user -n usr2 -p 654321 -P 2 -e wsnd
login user -n adm1n -p 123456
create team -n t1
login user -n usr -p 654321
join -n t1
login user -n usr2 -p 654321
join -n t1
login user -n adm1n -p 123456
admin load /home/ctf/flag
END
```

## 方法总结

- 核心技巧：把二进制当作一门小 DSL 解释器来逆向，先恢复 token 与语法，再找特权命令的语义约束。
- 识别信号：服务端收集一整段脚本后再交给本地程序执行，且二进制中出现大量 flex/yacc 风格 token 时，应优先分析语法动作和命令状态机。
- 复用要点：不要只盯内存破坏；这类 Pwn 题常把“漏洞”藏在解释器业务逻辑中。先确认服务端包装层是否只是传参，再在核心二进制中定位特权命令、当前用户、队伍成员数量等状态字段。
