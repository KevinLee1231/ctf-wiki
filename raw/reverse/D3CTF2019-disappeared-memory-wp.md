# disappeared_memory

## 题目简述

题目原方向为 Reverse，但主要障碍是 Windows 10 kernel dump 和 compressed memory 分析。可疑进程为 `D3CTF.exe`，代码逻辑会递归加密桌面上的 `*.png` 文件；代码段第一页可见，核心函数可通过 `LoadLibrary`/`GetProcAddress` 解析，但加密后的数据段无法直接访问。关键是参考 FireEye 对 Windows 10 Compressed Memory 的分析，从压缩内存中恢复缺失页面，再拿到加密数据进行还原。

题目资料地址：https://github.com/Ch111p/d3ctf2019_disappeared_memory

`D3CTF.c` 会递归搜索 `C:\Users\ch1p\Desktop\` 下的 `.png` 文件，文件大于 `0x100000` 时跳过。加密逻辑为 `srand(0x77777777)` 后逐字节 `mem[i] ^= (__int8)rand()`，随后删除原文件并以 `wb+` 写回。函数名通过 `GetProcAddress(GetModuleHandle("msvcrt"), "...")` 动态解析，这也解释了为什么 dump 中代码页可见但部分符号和数据页不完整。

## 解题过程

这道题是我在看blackhat 19年的议题时 看到了fireeye团队的

`https://i.blackhat.com/USA-19/Thursday/us-19-Sardar-Paging-All-Windows-Geeks-Finding-Evil-In-Windows-10-Compressed-Memory.pdf`

`https://i.blackhat.com/USA-19/Thursday/us-19-Sardar-Paging-All-Windows-Geeks-Finding-Evil-In-Windows-10-Compressed-Memory-wp.pdf`

这两篇 BlackHat 资料的主要信息是：Windows 10 引入 compressed memory 后，部分冷页不会以传统 pagefile 形式出现，而会进入 Store Manager 管理的 Virtual Store；分析时要从目标虚拟地址的 PTE 出发，判断页面是否被压缩并追踪 Store Manager 结构。这个机制很适合改成实践题，所以出了这道题。

考点如下：

首先是对于dump文件的分析 -> windbg的使用

可执行程序为D3CTF.exe 所以选手可以很快的定位，且dump出可执行程序的代码段

稍加分析可以看出来 做的事情就是在桌面上递归去用xor加密\*.png文件

这里值得一提的是代码段上只能看到第一页的数据 但是主体函数都在这里 只是函数符号出不来了..为了解决这个问题 这里函数调用都是用LoadLibrary、GetProcAddress获取(现在想来可能是这几页被放在了虚拟内存里面)

但是当选手想去访问数据段(加密后的数据) 会发现并不能访问到

结合上述背景继续查 compressed memory 的实现细节，可以定位到 FireEye 的 Virtual Store 深入分析。该文的关键不是题目外的额外步骤，而是给出了从 PTE 到压缩页内容的完整取证路线。

在本题环境里，需要调整的主要是最后一个结构偏移：Windows 10 1709 上对应偏移为 1848，附近候选指针不多，可以通过指针有效性和解压结果筛选。其余核心步骤是先计算/读取压缩页 key，再沿 Store Manager 的 B+ tree、store entry、chunk metadata 找到压缩数据，最后按记录的压缩算法解出原始 0x1000 页。

Windows 10 的 compressed memory 不会像传统 pagefile 那样把缺页内容完整落盘，而是把冷页压缩后放进由 Store Manager 管理的 Virtual Store，内存取证工具如果只遍历物理页和 pagefile，就会漏掉这些压缩页。FireEye 的思路是从目标进程虚拟地址入手，先检查 PTE 判断页面是否在 compressed store，再通过 `nt!SmGlobals`、Store Manager、B+ tree / store entry 等结构定位压缩数据所在位置，最后按记录的压缩格式解压还原页面内容。放到本题里，就是先定位 `D3CTF.exe` 中访问失败的数据页，再沿 compressed memory 结构把加密后的 PNG 数据恢复出来。

所以到最后也没有解我是蛮意外的 现在想来可能是一开始的description给的不太明显吧..但是里面其实也指出了可疑进程以及windows10的新特性(

也有可能是由于代码段后面几个页面也看不到 对选手造成了迷惑

最后给出的 hint 已经比较直接，复现时应优先沿 compressed memory 方向继续排查。

## 方法总结

- 核心技巧：不要只按普通内存 dump 搜进程地址空间；当 Windows 10 dump 中数据页缺失时，要检查 compressed memory 机制并恢复被压缩的页面。
- 识别信号：代码段部分可见、数据段访问失败、题目提示 Windows 10 新特性或 BlackHat compressed memory 资料时，应转向压缩内存取证。
- 复用要点：先用 WinDbg 定位可疑进程和代码逻辑，再按压缩内存文章恢复页面；文章中的结构偏移可能随 Windows 版本变化，实际分析时要在附近候选指针中验证。

