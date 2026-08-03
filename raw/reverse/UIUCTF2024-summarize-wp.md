# UIUCTF 2024 Summarize

## 题目简述

程序读取六个九位正整数 `a` 到 `f`，通过八组算术、位运算和取模条件检查输入。校验成功后，它使用：

```c
sprintf(buf, "uiuctf{%x%x%x%x%x%x}", a, b, c, d, e, f);
```

把六个整数分别转换为无前缀、无分隔符的小写十六进制并拼成 flag。发布二进制剥离了符号，校验函数又调用了多个用逐位循环实现的基础运算，直接符号执行容易发生状态爆炸。

## 解题过程

从 `__libc_start_main` 的参数定位 `main` 后，可以确认六个值必须满足：

```text
100000000 < a,b,c,d,e,f < 1000000000
```

`main` 调用地址 `0x40137b` 处的校验函数。反编译后把五个辅助函数分别识别为 `add`、`subtract`、`multiply`、`xor` 和 `and`，校验逻辑即可还原为：

```c
c1 = add(subtract(a, b), c) % 17492321;
c2 = add(a, b) % 17381917;
c3 = subtract(multiply(3, a), multiply(2, b)) % xor(a, d);
c4 = and(b, add(c, a)) % 28194;
c5 = add(b, d) % a;
c6 = xor(c, add(d, f)) % 1893928;
c7 = subtract(e, f) % 18294018;
c8 = add(e, f) % 48328579;

return c1 == 4139449   && c2 == 9166034   &&
       c3 == 556569677 && c4 == 12734     &&
       c5 == 540591164 && c6 == 1279714   &&
       c7 == 17026895  && c8 == 23769303;
```

这些辅助函数的结果很简单，但实现并不简单。例如 `add` 逐位模拟全加器，循环何时结束取决于符号输入。若让 angr 原样进入这些循环，它必须处理大量分支和不断变化的循环次数。解决办法是给每个函数写一个语义等价、没有循环的 `SimProcedure`，也就是函数摘要：

```python
import angr
import claripy
from angr.sim_type import SimTypeFunction, SimTypeInt


class Add(angr.SimProcedure):
    def run(self, a, b):
        return a + b


class Subtract(angr.SimProcedure):
    def run(self, a, b):
        return a - b


class Multiply(angr.SimProcedure):
    def run(self, a, b):
        return a * b


class XOR(angr.SimProcedure):
    def run(self, a, b):
        return a ^ b


class AND(angr.SimProcedure):
    def run(self, a, b):
        return a & b
```

随后建立六个 32 位符号变量，把发布二进制中对应函数地址替换为摘要：

```python
proj = angr.Project("./summarize", auto_load_libs=False)
prototype = SimTypeFunction(
    args=[SimTypeInt(), SimTypeInt()],
    returnty=SimTypeInt(),
)

proj.hook(0x40163D, Add(prototype=prototype))
proj.hook(0x4016D8, Subtract(prototype=prototype))
proj.hook(0x4016FE, Multiply(prototype=prototype))
proj.hook(0x40174A, XOR(prototype=prototype))
proj.hook(0x4017A9, AND(prototype=prototype))

a, b, c, d, e, f = [claripy.BVS(name, 32) for name in "abcdef"]
state = proj.factory.call_state(0x40137B, a, b, c, d, e, f)
simgr = proj.factory.simulation_manager(state)
simgr.explore(
    find=0x401628,
    avoid=[0x4013D2, 0x401412, 0x40162F],
)

found = simgr.found[0]
for value in (a, b, c, d, e, f):
    print(found.solver.eval(value))
```

`find` 是所有比较成功后的返回路径，三个 `avoid` 地址对应范围检查或条件检查失败。求得：

```text
a = 705965527
b = 780663452
c = 341222189
d = 465893239
e = 966221407
f = 217433792
```

把六个数提交给发布二进制，已验证程序输出：

```text
Correct.
uiuctf{2a142dd72e87fa9c1456a32d1bc4f77739975e5fcf5c6c0}
```

## 方法总结

本题名称提示了核心技术：函数摘要不是忽略被调用函数，而是用更紧凑、语义等价的模型替换它们。先通过逆向确认五个逐位循环确实等价于 32 位加、减、乘、异或和与，再让 angr 只求解真正有价值的八组约束，就能避开循环导致的状态爆炸。

最终还要注意 flag 不是六个十进制数本身，而是六次 `%x` 输出直接拼接；`%x` 不会为最后一个数补前导零，应该以程序实际输出为准。
