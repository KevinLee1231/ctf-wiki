# calcql

## 题目简述

服务随机生成大量 Python 函数，并为源码建立 CodeQL 数据库。每个函数只返回若干整数常量与其它函数调用的和；`entry_<随机哈希>` 函数分别代表 $1$ 到 $1023$ 的不同结果。玩家需提交一条 CodeQL 查询，在两套独立随机数据库中都找出返回值为 $42$ 的入口函数。

服务只从 CodeQL 输出最后一行提取 `entry_[0-9a-f]{20}`，并与各数据库私下保存的正确函数名比较，因此不能硬编码随机名称。

## 解题过程

### 把返回值定义为递归语义

生成器产生的函数形如：

```python
def fn_a1b2...():
    return 5 + 2 + 2

def entry_c3d4...():
    return fn_a1b2...() + 1 + 1
```

函数的数值等于它内部所有整数常量之和，加上它调用的每个函数的数值。生成器只引用此前已经生成的辅助函数，因此调用关系是无环的，可以用递归 CodeQL predicate 表达。

### 使用递归聚合

CodeQL 默认会限制递归聚合；`language[monotonicAggregates]` 允许这里使用单调的 `sum`。查询定义 `dcount(Function f)`：

```ql
import python

language[monotonicAggregates]

int dcount(Function f) {
  result =
    sum(IntegerLiteral i |
      f.contains(i)
    |
      i.getValue()
    ) +
    sum(Call c, Function callee |
      c.getFunc().(Name).toString() = callee.getName() and
      f.contains(c)
    |
      dcount(callee)
    )
}
```

第一项统计当前函数语法范围内的整数常量；第二项按调用表达式的名称连接到对应 `Function`，递归加入被调函数的值。

最后只筛选入口函数并要求结果等于 $42$：

```ql
from Function f
where
  f.getName().prefix(6) = "entry_" and
  dcount(f) = 42
select f.getName(), dcount(f)
```

同一查询会在两套随机生成的数据库中分别计算语义结果，输出各自正确的随机入口名，服务验证通过后返回 flag。

## 方法总结

- 核心技巧：用 CodeQL 递归 predicate 对调用图做跨过程常量求值，并开启单调聚合支持。
- 识别信号：大量随机命名函数组成简单表达式 DAG，目标依赖函数返回值而不是名字或源码位置。
- 复用要点：静态查询应表达稳定的程序语义；连接调用点与函数定义时需明确名称和作用域，真实项目中还应考虑重名、动态调用和递归环。本题生成器保证全局随机名与无环依赖，名称连接足够可靠。
