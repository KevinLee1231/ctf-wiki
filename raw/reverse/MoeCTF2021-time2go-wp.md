# time2go

## 题目简述

附件是 Go 编译的可执行文件。程序会逐字输出 flag，但每轮都调用 `time.Sleep`，累计运行时间超过 55 秒后便进入失败分支；即使绕过计时，正常循环也只会给出 flag 的前半部分。

这题的两个关键点是识别 Go 二进制中的运行时函数，以及找到藏在不可达分支中的剩余字符串。

## 解题过程

Go 二进制通常保留较有辨识度的函数名。载入 IDA 后，可以从 `main.main`、`main.fun1`、`main.fun2` 和 `time.Sleep` 等符号快速定位核心逻辑。整理后的主循环如下：

```go
start := time.Now()
for i := 0; i < 20; i++ {
    timer := i << 2
    fmt.Print(fun2(i))
    time.Sleep(time.Duration(timer%15) * time.Second)

    if time.Since(start).Seconds() > 55 {
        fmt.Println(wrong)
        os.Exit(114514)
    }
}
```

每轮等待时间为 $(4i)\bmod 15$ 秒，累计时间必然触发失败条件。可以在副本中把循环内对 `time.Sleep` 的调用改为 NOP，或者把传入的持续时间改成 0。重新运行后，程序快速完成 20 轮并输出：

```text
moectf{G0_1an8uag3_1
```

`fun1` 使用数组 `egg` 对一个 256 项表进行类似 RC4 KSA 的置换，结果保存在全局变量 `Box` 中；`fun2` 再用它解码 `var1`：

```go
var egg = []int{104, 115, 121, 121, 100, 115}
var var1 = []int{
    167, 38, 65, 71, 210, 47, 177, 14, 20, 123,
    151, 40, 164, 113, 81, 69, 193, 122, 149, 120,
}

func fun2(a int) string {
    if a < 0 {
        fmt.Print(CanuFindme)
    }
    t := var1[a] ^ Box[egg[a%6]]
    return string(t)
}
```

正常循环只会以 `0` 到 `19` 调用 `fun2`，所以 `a < 0` 分支不可达。跟踪该分支引用的全局字符串 `CanuFindme`，可直接读到：

```text
5_amaz1ng}
```

不能简单地强制调用 `fun2(-1)`：打印字符串后函数仍会继续访问 `var1[-1]`，从而触发越界。静态读取全局字符串，或在打印后直接返回，才是稳定的处理方式。

拼接两部分后得到：

```text
moectf{G0_1an8uag3_15_amaz1ng}
```

## 方法总结

分析 Go 程序时，运行时符号和包路径往往比大段反编译伪代码更有价值。本题先沿 `time.Sleep` 解除人为延时，再根据 `main.fun2` 的数据流检查正常循环无法到达的分支，就能分别取得 flag 的两部分。补丁应只施加在附件副本上，并在修改前记录原始控制流，避免把补丁后的行为误当成原程序逻辑。
