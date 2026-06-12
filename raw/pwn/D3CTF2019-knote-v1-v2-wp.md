# knote (v1, v2)

## 题目简述

题目是 kernel pwn。v1 是环境打包失误导致文件 owner 不正确，可通过替换 `/bin/umount` 在 init 脚本以 root 卸载 proc 时劫持执行流。v2 才是主要预期漏洞：`edit` 和 `get` 功能缺少锁，存在竞争条件；利用 `userfaultfd` 稳定卡住 `copy_to_user`/`copy_from_user` 流程形成 double fetch，释放 note 后用 `tty_struct` 占位，泄露内核 code/heap 地址并在同一竞态窗口篡改 `tty_operations`，最后绕过 SMEP/SMAP 提权。

## 解题过程

v1的话其实问题就是直接把测试环境打包忘记改文件的owner了，导致选手可以直接删改任意文件，半夜发现有人一血了看了眼log我都傻了（太菜了），原理是init脚本里最后以root运行umount卸载proc文件系统，可以劫持umount命令为sh，例如如下exp

```bash
rm /bin/umount
echo "#!/bin/sh" > /bin/umount
echo "/bin/sh" >> /bin/umount
exit
```

v2:  
洞：edit和get功能没上锁，存在竞争，但直接竞争还是有很大困难的，所以利用userfaultfd稳定double fetch

先add一个note，然后设置一个userfaultfd给ptr之后进行get(同时进行edit用来待会修改内核数据)，线程就会卡在user\_copy，此时在fault\_handle里dele掉note，然后申请tty\_struct就有机会申请到这个note里，然后处理缺页异常将数据带回用户态，这里由于copy的顺序问题无法控制头0x20字节，所以leak就得用0x250偏移处的其他指针了，不过问题不大

leak出code地址和heap地址

然后leak成功的同时有了一次写的机会，赶紧把刚刚获得的数据作为缺页拷贝的数据，但是篡改其中的ops就可以控制程序流

开始绕过+smep,+smap  
我这里是进行了physmap喷射，然后劫持ioctl进行栈迁移

这里可控的寄存器只有rdx（和它的几个拷贝），rsi只可控低32位作用不大，注意到rbp指向tty\_struct，然后ioctl只需要check tty->drive和magic等几个属性而已，其他的都可以改，所以可以改+0x48 +0x58的数据进行jop

一开始先跳这里  
`0xffffffff8135b69a: push rdx; fdiv st(7), st(0); call qword ptr [rbp + 0x48];`  
然后rbp+0x58设置成下面这条  
`0xffffffff81836504: jmp qword ptr [rbp + 0x58];`  
就会死循环原地跳  
然后乘机用另一个线程  
把+0x58的改成  
`0xffffffff81092e0b: pop rsi; jmp qword ptr [rbp + 0x48];`  
把+0x48的改成  
`0xffffffff810027ce: pop rsp; ret;`

整个流程像下面这样

```
ioctl->0xffffffff8135b69a push rdx; fdiv st(7), st(0); call qword ptr [rbp + 0x48];

thread1:
rbp+0x58 0xffffffff81836504 jmp qword ptr [rbp + 0x58]; <loop>
rbp+0x48 0xffffffff81836504 jmp qword ptr [rbp + 0x58]; <loop>

thread2:overwrite rbp+0x58 & rbp+0x48
rbp+0x58 0xffffffff81092e0b pop rsi; jmp qword ptr [rbp + 0x48]; 
rbp+0x48 0xffffffff810027ce pop rsp; ret; <control the rsp to rops>
```

然后就是提权的rop和iretq了，没啥好说的

另外xdssll的师傅和国外badfirmware战队后面使用了其他更简单的方法，后者应该没写wp，前者可以看看他自己公布的wp

## 方法总结

- 核心技巧：用 `userfaultfd` 把用户态缺页变成可控同步点，稳定扩大内核 double fetch/竞态窗口，再用 `tty_struct` 作为可预测的内核对象完成泄露和控制流劫持。
- 识别信号：kernel 模块的 get/edit 路径没有锁、涉及 user copy、对象释放后可重新分配时，应考虑 `userfaultfd` 卡住 copy 并配合 UAF 对象占位。
- 复用要点：绕过 SMEP/SMAP 时要关注 ioctl 调用现场可控寄存器和结构体字段；如果只能部分控制寄存器，可以通过 JOP 原地循环等待另一线程二次覆盖，再切到栈迁移和 ROP。

