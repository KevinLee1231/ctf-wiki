# hackphp

## 题目简述

本题给出 PHP 扩展 `hackphp.so`，扩展中定义了 `vline` 类。创建大于 `0x200` 或小于 `0x100` 的 buffer 会触发 UAF；由于题目允许较自由地申请 chunk，可以把 PHP 对象重新分配到可控区域，再通过 `hackphp_edit` 修改对象元数据获得读写能力。

题目附件包含 PHP 扩展源码/二进制和 exp，核心对象是 `vline` 类以及触发 UAF 的 buffer size 逻辑。主利用路线是构造 leak 原语找到可控 string 对象地址，在其中伪造 `zend_closure`，再借 UAF 把 closure 指向伪造对象执行。

## 解题过程

题目 exp 位于 [`UESuperGate/D3CTF-2021-Exploits`](https://github.com/UESuperGate/D3CTF-2021-Exploits)。下文保留了 exp 的核心结构：`vline` 类 UAF、受限 chunk 内伪造对象、泄露可控 string 地址和 `zend_closure` 伪造。

用 IDA 分析 `hackphp.so` 可以看到两个关键点：

- 扩展定义了一个名为 `vline` 的类。
- 创建大于 `0x200` 或小于 `0x100` 的 buffer 会导致 UAF。

考虑到实际上可以申请任意大小的 chunk，并且很容易造成 UAF，可以将一个对象通过 UAF 分配到完全可控的空间内，然后用 `hackphp_edit` 函数修改对象的相关信息，从而实现任意读写。

总体思路和前述 `php7-backtrace-bypass` exp 类似，有些模板函数可以直接复用；但本题中只能控制当前 `size` 范围内的区域，无法越界读写到下一个堆块，这给构造 `zend_closure` 带来了额外限制。

解决思路是再次申请一个 string 类型对象，长度至少要能容纳 `0x130` 字节的 `zend_closure` 内容，并在其中填满特征值。随后通过构造出的 leak 原语找到该 string 对象地址，在 string buffer 中伪造 closure，最后通过 UAF 把 closure 指针改到伪造地址。

extension 里内置了 `vline` 类，降低了构造对象和触发 UAF 的难度；即使没有该类，也可以在用户态定义结构相近的类尝试复用同一思路。

存在 UAF 后并不一定要走伪造对象路线，本题也可以尝试两类替代思路：

- 通过 `LD_PRELOAD` 构造恶意 `.so`，原因是黑名单没有禁掉 `putenv` 这类危险函数，属于更偏 Web 的做法。
- 在 PHP 堆空间上做堆风水，把可控对象或字符串布置到更适合伪造结构的位置。

题目 exp 的关键是把 PHP 用户态对象布局、string buffer 和伪造 closure 关联起来；先泄露可控字符串地址，再把 UAF 对象解释为伪造 `zend_closure`。伪造 closure 的思路可参考 [php7-backtrace-bypass exploit](https://github.com/mm0r1/exploits/blob/master/php7-backtrace-bypass/exploit.php)。

## 方法总结

- 核心技巧：PHP 扩展 UAF 到对象伪造，通过可控 chunk、泄露原语和伪造 `zend_closure` 完成代码执行。
- 识别信号：扩展暴露自定义类和编辑函数，特定大小范围的 buffer create 导致对象生命周期异常，且只能控制当前 chunk 的 `size` 范围。
- 复用要点：PHP UAF 不一定要越界读写相邻块；可以另申请足够大的 string 承载伪造结构，先 leak 该 string 地址，再把悬垂引用改到伪造对象。

