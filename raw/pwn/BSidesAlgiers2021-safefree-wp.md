# safefree

## 题目简述

safefree 是 amd64、No PIE、Partial RELRO、NX、Canary 的 glibc 2.27 堆题。程序宣传的安全释放函数本身没有问题：

~~~c
void safefree(void **pp) {
    free(*pp);
    *pp = NULL;
}
~~~

漏洞出在调用者：

~~~c
safefree((void **)chunks[idx]);
~~~

这里传入的是 chunk 内容地址，而不是保存指针的数组槽地址。于是 chunk 开头的 8 字节会被当作待 free 的指针，形成一次攻击者可控的 arbitrary free，随后该 8 字节被清零。

## 解题过程

官方 exploit 先通过 glibc 2.27 的 tcache/unsorted-bin 残留数据取得地址：

1. 分配并释放两个 0x10 请求，重新分配后用 view 读取 tcache 指针，页对齐得到 heap base。
2. 分配 9 个 0x80 请求并全部释放，使 tcache 填满且额外 chunk 进入 unsorted bin。
3. 再利用重叠的空闲内容读取 main_arena 指针；对题目 libc 减去 0x3ebca0 得到 libc base。

接着在堆上布置伪 0x21 大小 chunk，并让一个正常 chunk 的前 8 字节保存该伪 chunk 的地址。调用一次试用版 safefree 时，错误的二级指针调用会执行：

~~~c
free(fake_chunk_address);
~~~

这把伪 chunk 放进对应 tcache。重新分配并改写其 next 指针后，可让后续分配落到 __free_hook - 8。官方脚本最终完成：

~~~text
目标 chunk: 00 00 00 00 00 00 00 00 | system 地址
普通 chunk: /bin/sh\0
~~~

释放保存“/bin/sh”的 chunk 后，__free_hook 把 free 调用转成 system 调用，获得 shell。读取结果为：

~~~text
shellmates{M4AAYb3_juSt_STIck_t0_tHE_OLD_WaYS}
~~~

源码中的索引断言同样使用 idx <= 16，理论上允许数组外的第 16 项，但官方利用链不需要依赖它。

## 方法总结

“释放后置 NULL”只有在参数真的是指针槽地址时才安全。若把用户缓冲区地址强制转换成 void**，函数就会把用户数据解释为地址并产生 arbitrary free。此类题应先区分“chunk 地址”“chunk 中保存的值”和“指向全局槽的地址”三个层次，再利用目标 glibc 的 tcache 与 unsorted-bin 元数据完成泄漏和 freelist 控制。
