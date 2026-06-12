# Smart Calculator

## 题目简述

本题包含两个进程。父进程从标准输入读取 solver_id , expression  和 result ，并通过消息队列传递给子进程；子进程从消息队列读取这三个参数，并计算 expression  的值，最后输出结果（即计算结果与 result  是否相等）。

漏洞链由两部分组成：一是比较失败时输出 `result` 触发越界读，可泄露子进程栈信息；二是父进程允许较长的 `result`，子进程拷贝到 264 字节栈缓冲区时触发栈溢出。结合消息队列单条消息长度限制，可以制造参数错位，把可控的 `solver_id` 当成 `result` 完成 ROP。

## 解题过程

注意到在结果不同时，会通过 write  输出 result  的值，此时会有一个越界读，能够 leak 子进程栈上的信息。

泄露点在结果比较失败后的输出逻辑：程序会把 `result` 作为字符串写回，而该区域和子进程栈上数据相邻，因此可以越界读出栈信息。

同时，我们发现在父进程，输入的 result  最长为 0X1f00，但是在子进程会将 result  拷贝到栈上缓冲区，但这个缓冲区只有 264 的大小，因此可以造成栈溢出。

溢出点在子进程读取后执行的拷贝：`expression` 栈缓冲区只有约 `264` 字节，但父进程允许传入最长 `0x1f00` 的 `result`，随后 `memcpy(..., 0x1f00)` 触发栈溢出。

但是由于 expression  和 result  都会检查是否为可见字符，无法直接在栈上构造 ROP 链，因此我们考虑在 solver_id  上构造 ROP，并且如果存在一个错位的漏洞，即让子进程将 solver_id  当作result  读进来，就可以很容易 getshell 了。根据 README.txt  和 hint 可以知道远程环境的

kernel.msgmax  参数为 8192，该参数定义了系统消息队列中每一个消息的最大长度。对于远程环境对应的 linux kernel 版本而言，如果通过 msgsnd  传入的消息长度大于 8192，则会导致消息入队失败，而 solver_id  最大长度可以到 8208，所以我们可以使得 solver_id  入队失败，然后导致消息错位，最后通过 ROP 拿到 flag。

```
father               child
solver_id ---xxx-->
expr      --------> solver_id
result    --------> expr
solver_id ---ROP--> result       Overflow
```

## 方法总结

- 核心技巧：利用 `kernel.msgmax=8192` 限制让超长 `solver_id` 入队失败，导致父子进程对消息队列中参数顺序的理解错位。
- 利用链：先通过错误输出的越界读泄露子进程栈信息，再用错位后的 `result` 栈溢出，把 ROP 放到不受可见字符限制影响的 `solver_id` 中。
- 复用要点：多进程 + IPC 题要同时检查生产者和消费者的长度限制；只要某一条消息发送失败但状态机继续推进，就可能出现参数错位。
