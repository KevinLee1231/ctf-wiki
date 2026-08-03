# weirdcaml

## 题目简述

附件 `main.ml` 把一个布尔可满足性问题编码进 OCaml 的 GADT 类型系统。104 个变量 `flag_000` 到 `flag_103` 表示 13 字节答案，其余变量是辅助量；上千个 `p*_t` 类型表示约束。直接编译会让 OCaml 的穷尽性检查器尝试构造反例，但更稳定的做法是解析 GADT 并交给 Z3 求解。

## 解题过程

题目先用 phantom type 表示真假：

```ocaml
type b_true
type b_false
type 'a val_t =
  | T : b_true val_t
  | F : b_false val_t
```

每个 `pN_t` 的构造器约束一组类型参数。例如某个构造器返回 `('a, b_true, 'c, 'd) pN_t`，就表示第二个布尔变量为真时，这个证明对象可以存在。一个 `pN_t` 有多个构造器，等价于这些可满足分支的逻辑或。巨大的 `Puzzle` 构造器把 277 个变量及 1435 个证明对象装进同一个存在类型，最后的 refutation case：

```ocaml
let check (f : puzzle) = function
  | Puzzle _ -> .
  | _ -> ()
```

迫使编译器判断 `Puzzle _` 是否不可达；若 SAT 实例存在，编译错误中的 witness 就能泄漏一组赋值。

自动求解时先从 `type puzzle = Puzzle : ...` 取出变量的实际顺序，再对每个 `pN_t` 做两次映射：局部类型参数映射到 `Puzzle` 中使用的实际变量，构造器返回类型中的 `b_true`、`b_false` 映射为 Z3 literal。核心逻辑可概括为：

```python
from z3 import And, Bool, Not, Or, Solver, sat

variables = {name: Bool(name) for name in parsed_variable_names}
clauses = []

for proof_type in parsed_proof_types:
    branches = []
    for constructor in proof_type.constructors:
        terms = []
        for actual_name, returned_type in constructor.bindings:
            if returned_type == "b_true":
                terms.append(variables[actual_name])
            elif returned_type == "b_false":
                terms.append(Not(variables[actual_name]))
        if terms:
            branches.append(And(*terms))
    clauses.append(Or(*branches))

# 最后一个一元证明类型是 puzzle 元组的末项，需单独处理：
# type ('a) p1435_t = P1435_1 : b_true val_t -> (b_false) p1435_t
# Puzzle 中实例化为 ('a57) p1435_t，因此 a57 必须为 false。
clauses.append(Not(variables["a57"]))

solver = Solver()
solver.add(*clauses)
assert solver.check() == sat
model = solver.model()
```

仓库中共有 277 个变量、104 个 flag bit 和 1435 个证明类型。前 1434 个是规则的四参数项；末尾 `p1435_t` 是 `Puzzle` 元组中没有尾随星号的一元末项，通用正则容易漏掉它。其唯一构造器把类型参数固定为 `b_false`，而 `Puzzle` 将它实例化为 `a57`，所以补入 `Not(a57)` 即得到完整约束集。按 `flag_000` 到 `flag_103` 排序，每 8 位以高位在前组成一个字符：

```python
bits = [
    1 if bool(model.eval(variables[f"flag_{i:03d}"])) else 0
    for i in range(104)
]
answer = bytes(
    int("".join(map(str, bits[i:i + 8])), 2)
    for i in range(0, 104, 8)
)
print(answer)
```

本地 Z3 复核输出：

```text
sat_on_a_caml
```

按题意补上外层格式：

```text
uiuctf{sat_on_a_caml}
```

## 方法总结

- 核心技巧：把 GADT 构造器“该证明对象何时存在”的类型条件还原成布尔公式，再用 SMT 求满足赋值。
- 识别信号：大量 phantom type、只携带类型信息的构造器，以及 `-> .` refutation case，常见于把逻辑命题交给编译器类型检查器的题目。
- 复用要点：解析时必须区分每个 GADT 声明中的局部参数名与 `Puzzle` 中的实际变量名；flag bit 的排序和位序也要单独验证，不能依赖字典的默认遍历顺序。
