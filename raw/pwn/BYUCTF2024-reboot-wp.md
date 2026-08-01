# Reboot

## 题目简述

服务允许设置最长 30 字符的 hostname；“重启”后把它直接插入 `os.system("cat /etc/hosts | grep ... -i")`。存在 shell 命令注入，但容器禁止外连，flag 路径未知，单条命令又受长度限制。

## 解题过程

利用可写且短路径的 `/dev/shm` 做两阶段传递。第一次 hostname 设置为：

```text
;find /o* -type f>/dev/shm/a;
```

触发重启后，分号结束 `grep` 参数，`find` 把根目录下 `/o*` 路径中的普通文件名写入 `/dev/shm/a`。Docker 布局使目标 flag 位于该搜索范围。

第二次设置：

```text
;cat $(cat /dev/shm/a);
```

再次重启，内层 `cat` 读回路径，外层 `cat` 输出目标文件。两条字符串都不超过 30 字符，也不需要反连或外部下载，最终得到：

```text
byuctf{expl0iting_th1s_r3al_w0rld_w4s_s000_ann0ying}
```

## 方法总结

短命令限制常可通过目标机本地文件分阶段绕过。先把枚举结果落到短路径，再用命令替换消费它；同时要根据容器文件布局缩小 `find` 范围，否则长度和输出噪声都会失控。
