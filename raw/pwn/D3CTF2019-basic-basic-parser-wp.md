# basic basic parser

## 题目简述

题目给出 C++ parser 源码，漏洞点隐藏在语法分析流程中。通过对关键 token 组合 fuzz，构造 `begin ... function a; ... end` 或 `integer function AAA;` 这类不完整函数声明，可触发 `Process` 对象被释放后仍保留在 `allProcess` 列表中。程序最后遍历该列表输出变量，形成 heap use-after-free；利用时可先覆盖字符串字段泄露地址，再伪造成员触发 double free/tcache attack，最终改写 GOT 或退出路径中的可控调用 getshell。

## 解题过程

这题其实比较难的地方就是找漏洞，漏洞找到了之后利用其实是非常简单的

因为题目给了源码，所以我们可以 ~~硬核c++代码审计~~ 借助addresssanitizer或者其他的一些工具来帮助我们分析

`clang++ prob.cpp -o probf -fsanitize=address -std=c++11 -g -O0 -fno-omit-frame-pointer -fsanitize-recover=address`

而在代码中可以找到一些关键词：

```c
void initTable()
{
        symbol.insert(pair<string, int>("begin", 1));
        symbol.insert(pair<string, int>("end", 2));
        symbol.insert(pair<string, int>("integer", 3));
        symbol.insert(pair<string, int>("if", 4));
        symbol.insert(pair<string, int>("then", 5));
        symbol.insert(pair<string, int>("else", 6));
        symbol.insert(pair<string, int>("function", 7));
        symbol.insert(pair<string, int>("read", 8));
        symbol.insert(pair<string, int>("write", 9));
        symbol.insert(pair<string, int>("symbol", 10));
        symbol.insert(pair<string, int>("constant", 11));
        symbol.insert(pair<string, int>("=", 12));  // eq
        symbol.insert(pair<string, int>("<>", 13)); // ne
        symbol.insert(pair<string, int>("<=", 14));
        symbol.insert(pair<string, int>("<", 15));
        symbol.insert(pair<string, int>(">=", 16));
        symbol.insert(pair<string, int>(">", 17));
        symbol.insert(pair<string, int>("-", 18));
        symbol.insert(pair<string, int>("*", 19));
        symbol.insert(pair<string, int>(":=", 20));
        symbol.insert(pair<string, int>("(", 21));
        symbol.insert(pair<string, int>(")", 22));
        symbol.insert(pair<string, int>(";", 23));
    }
```

可以简单的对这些关键词进行排列组合来对整个程序进行fuzz，当在 `begin` 和 `end` 中间构造出形如 `function a;`或者 `integer function AAA;`这样的形式时，会触发crash并下面这样的输出：

```yaml
==4734==ERROR: AddressSanitizer: heap-use-after-free on address 0x607000005b58 at pc 0x00000043afde bp 0x7ffd4064c510 sp 0x7ffd4064bcc0
READ of size 2 at 0x607000005b58 thread T0
    #0 0x43afdd  (/home/pzhxbz/Desktop/p/probf+0x43afdd)
    #1 0x52a278  (/home/pzhxbz/Desktop/p/probf+0x52a278)
    #2 0x531b7f  (/home/pzhxbz/Desktop/p/probf+0x531b7f)
    #3 0x537afe  (/home/pzhxbz/Desktop/p/probf+0x537afe)
    #4 0x51bfc1  (/home/pzhxbz/Desktop/p/probf+0x51bfc1)
    #5 0x5195f0  (/home/pzhxbz/Desktop/p/probf+0x5195f0)
    #6 0x7f30750f7b96  (/lib/x86_64-linux-gnu/libc.so.6+0x21b96)
    #7 0x41c509  (/home/pzhxbz/Desktop/p/probf+0x41c509)

0x607000005b58 is located 56 bytes inside of 80-byte region [0x607000005b20,0x607000005b70)
freed by thread T0 here:
    #0 0x5156e8  (/home/pzhxbz/Desktop/p/probf+0x5156e8)
    #1 0x530cd7  (/home/pzhxbz/Desktop/p/probf+0x530cd7)
    #2 0x52f33c  (/home/pzhxbz/Desktop/p/probf+0x52f33c)
    #3 0x52e0ab  (/home/pzhxbz/Desktop/p/probf+0x52e0ab)
    #4 0x52c4e6  (/home/pzhxbz/Desktop/p/probf+0x52c4e6)
    #5 0x51b8e0  (/home/pzhxbz/Desktop/p/probf+0x51b8e0)
    #6 0x519305  (/home/pzhxbz/Desktop/p/probf+0x519305)
    #7 0x7f30750f7b96  (/lib/x86_64-linux-gnu/libc.so.6+0x21b96)

previously allocated by thread T0 here:
    #0 0x514970  (/home/pzhxbz/Desktop/p/probf+0x514970)
    #1 0x53095a  (/home/pzhxbz/Desktop/p/probf+0x53095a)
    #2 0x52f33c  (/home/pzhxbz/Desktop/p/probf+0x52f33c)
    #3 0x52e0ab  (/home/pzhxbz/Desktop/p/probf+0x52e0ab)
    #4 0x52c4e6  (/home/pzhxbz/Desktop/p/probf+0x52c4e6)
    #5 0x51b8e0  (/home/pzhxbz/Desktop/p/probf+0x51b8e0)
    #6 0x519305  (/home/pzhxbz/Desktop/p/probf+0x519305)
    #7 0x7f30750f7b96  (/lib/x86_64-linux-gnu/libc.so.6+0x21b96)

SUMMARY: AddressSanitizer: heap-use-after-free (/home/pzhxbz/Desktop/p/probf+0x43afdd) 
Shadow bytes around the buggy address:
  0x0c0e7fff8b10: 00 00 00 00 00 00 00 00 00 fa fa fa fa fa 00 00
  0x0c0e7fff8b20: 00 00 00 00 00 00 00 fa fa fa fa fa 00 00 00 00
  0x0c0e7fff8b30: 00 00 00 00 00 fa fa fa fa fa 00 00 00 00 00 00
  0x0c0e7fff8b40: 00 00 00 fa fa fa fa fa 00 00 00 00 00 00 00 00
  0x0c0e7fff8b50: 00 fa fa fa fa fa 00 00 00 00 00 00 00 00 00 00
=>0x0c0e7fff8b60: fa fa fa fa fd fd fd fd fd fd fd[fd]fd fd fa fa
  0x0c0e7fff8b70: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c0e7fff8b80: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c0e7fff8b90: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c0e7fff8ba0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c0e7fff8bb0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca rdzone:     ca
  Right alloca redzone:    cb
==4734==ABORTING
```

跟着报错信息，可以找到这个地方：

```c
lastProcess = nowProcess;
 nowProcess = new Process(p->name, lastProcess->getLevel() + 1);
 allProcess.push_back(nowProcess);
 addVar(p->name, FUNCTION, 1);
 auto ret = S(_get_next(nnext));
 if(ret == _get_next(nnext))
 {
     delete(nowProcess);
     nowProcess = lastProcess;
     return nnext;
 }
 nowProcess = lastProcess;
 return ret;
```

这里在没有获取到function的主体时会直接free掉当前的process，但是在这之前已经被放入了  
allProcess这个list中，而在最后输出变量的时候会遍历这个list，导致了uaf。

利用的话其实没什么难度，就是个常规的uaf，首先覆盖process类中的securt或者processName这些字符串来leak，然后再伪造vars成员来double free，之后直接tcache attack，为了增加利用成功率，这个程序里面got表可写，最后退出的时候也送了一个可控的调用，直接就可以getshell。

## 方法总结

- 核心技巧：源码 parser 题先用 grammar token 组合 fuzz 找异常路径，再借 ASan 定位释放后仍被全局容器引用的对象。
- 识别信号：对象在错误恢复路径中 `delete`，但此前已插入全局 list/vector，且后续还会统一遍历输出时，应重点检查 UAF。
- 复用要点：C++ 类对象 UAF 常能先覆盖 `std::string` 或指针成员泄露，再伪造内部容器成员制造二次释放；若 GOT 可写且退出阶段存在可控调用，利用链可以明显简化。

