# UIUCTF 2023 Corny Kernel Writeup

## 题目简述

题目提供一个可加载内核模块 `pwnymodule`，并给出其源码。模块引用内核内置的两个导出符号 `flag1` 和 `flag2`：初始化回调打印前半段 flag，退出回调打印后半段 flag。

```c
static int __init pwny_init(void)
{
    pr_alert("%s\n", flag1);
    return 0;
}

static void __exit pwny_exit(void)
{
    pr_info("%s\n", flag2);
}
```

## 解题过程

连接远端 QEMU shell 后，先确认题目提供的模块文件名。比赛环境中模块以 `pwnymodule.ko.gz` 形式出现，可直接加载：

```bash
insmod pwnymodule.ko.gz
```

`insmod` 调用模块的 `module_init` 回调，第一段内容通过 `pr_alert` 写入内核日志。随后卸载模块：

```bash
rmmod pwnymodule
```

这会触发 `module_exit` 回调，把第二段内容通过 `pr_info` 写入日志。最后查看最近的内核消息：

```bash
dmesg | tail -n 10
```

两条日志依次给出：

```text
uiuctf{m4ster_
k3rNE1_haCk3r}
```

拼接后得到：

```text
uiuctf{m4ster_k3rNE1_haCk3r}
```

## 方法总结

这是一道内核模块生命周期入门题。`module_init` 在加载时执行，`module_exit` 在卸载时执行，而 `pr_alert`、`pr_info` 等宏把内容写入 kernel ring buffer，可通过 `dmesg` 读取。审计模块时不能只观察加载阶段；退出回调、错误回滚路径和日志往往也承载关键行为。
