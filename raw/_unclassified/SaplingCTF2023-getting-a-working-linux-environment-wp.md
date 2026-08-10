# Getting a Working Linux environment

## 题目简述

这是一道环境准备检查题。附件 flag 是 Linux 可执行文件，题目明确说明没有隐藏技巧，只要在可运行 ELF 的 Linux 环境中赋予执行权限并启动即可。它不要求逆向程序行为，因此不应仅因附件是二进制就归入 Reverse。

## 解题过程

先确认文件类型并赋予执行权限：

~~~bash
file flag
chmod +x flag
./flag
~~~

程序直接打印：

~~~text
maple{this_1s_a_fl4g}
~~~

Windows 用户可在 WSL、Linux 虚拟机或原生 Linux 中执行；macOS 不能直接运行 Linux ELF，需要使用兼容的 Linux 环境。

## 方法总结

文件扩展名不是判断可执行格式的依据，file 和 ELF 头才是证据。运行失败时应依次检查执行权限、体系结构与动态加载器，而不是立刻进入逆向。本题的决定性任务只是准备环境，保留在 _unclassified 比强行归类更准确。
