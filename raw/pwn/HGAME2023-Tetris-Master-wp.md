# Tetris Master

## 题目简述

题目通过 SSH 登录后自动进入一个 Bash 编写的俄罗斯方块游戏。预期路线是正常游玩并累计分数，但初版部署没有正确限制玩家退出前台游戏，因此可以直接从游戏返回到交互式 shell。

## 解题过程

SSH 会话中，俄罗斯方块只是当前 shell 启动的前台进程。程序没有捕获或屏蔽终端发送的 `SIGINT`，所以按下 `Ctrl+C` 后，游戏进程被中断，但承载会话的 shell 仍然存在。

```text
Are you tetris master?[y/n]
^C
ctf@gamebox:~$
```

取得 shell 后直接读取根目录中的 flag：

```bash
cat /flag
```

得到：

```text
hgame{Bash_Game^Also*Can#Rce}
```

这个入口属于初版部署的非预期行为，随后题目降低分值并上线了 Revenge 版本。

## 方法总结

通过 SSH、容器终端或受限 shell 启动交互程序时，需要区分“终止前台程序”和“终止整个会话”。若包装层仍保留普通 shell，`Ctrl+C`、挂起、异常退出等路径都可能把玩家直接送回命令行。审计这类题时，应先检查信号处理、父子进程关系和退出后的控制流。
