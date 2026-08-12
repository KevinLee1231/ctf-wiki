# Co-Program

## 题目简述

题目包含两部分，均使用 36 位无符号整数。Co-Login 给出 100 个含变量的算术表达式及期望结果，需在两分钟内为至少 90 个表达式反求变量；Co-UnitTest 给出 5 组 $(x,y,z)$ 样例，要求在 30 秒内合成只含变量 `x`、`y`、不含整数常量的表达式，10 轮中完成 7 轮。

运算按模 $2^{36}$ 执行。除数为 0 时，除法结果为 $2^{36}-1$，取模结果为被除数；这与 SMT-LIB 位向量的无符号除法/余数语义一致。

## 解题过程

### Co-Login：翻译为 36 位 SMT

不要用数学整数模拟再手工补溢出，而是把每个变量直接声明为 36 位 `BitVec`。将题目解析器产生的 AST 递归翻译：

```python
import z3

WIDTH = 36

def to_z3(node, variables):
    kind = node.kind
    if kind == "number":
        return z3.BitVecVal(node.value, WIDTH)
    if kind == "variable":
        return variables.setdefault(node.name,
                                    z3.BitVec(node.name, WIDTH))
    if kind == "neg":
        return -to_z3(node.child, variables)

    left = to_z3(node.left, variables)
    right = to_z3(node.right, variables)
    return {
        "add": lambda: left + right,
        "sub": lambda: left - right,
        "mul": lambda: left * right,
        "div": lambda: z3.UDiv(left, right),
        "mod": lambda: z3.URem(left, right),
    }[kind]()
```

对每轮添加 `to_z3(expression) == expected`，求解后输出模型中的所有变量。未出现在模型中的自由变量可填 0。Z3 位向量天然截断到 36 位，并且 `UDiv(x, 0)`、`URem(x, 0)` 正好符合题目约定，无需增加分支。

```python
solver = z3.Solver()
variables = {}
solver.add(to_z3(ast, variables) == z3.BitVecVal(expected, 36))

if solver.check() == z3.sat:
    model = solver.model()
    answer = {
        name: model.eval(var, model_completion=True).as_long()
        for name, var in variables.items()
    }
```

通过至少 90 轮后得到形如 `flag{z3isgood!-...}` 的第一段 flag。

### Co-UnitTest：SyGuS 程序合成

第二部分是标准 Syntax-Guided Synthesis。为 CVC5 声明 36 位变量和待合成函数 `f`，语法只允许题目支持的算术、比较、布尔连接与 `ite`，然后把五组样例写成约束：

```lisp
(set-logic BV)
(synth-fun f ((x (_ BitVec 36)) (y (_ BitVec 36))) (_ BitVec 36)
  ((I (_ BitVec 36)
      (x y
       (bvneg I) (bvadd I I) (bvsub I I) (bvmul I I)
       (bvudiv I I) (bvurem I I) (ite B I I)))
   (B Bool
      (true false (= I I) (bvule I I)
       (not B) (and B B) (or B B)))))
; 对每组样例添加 (constraint (= (f #x... #x...) #x...))
(check-synth)
```

CVC5 返回表达式后，把 `bvadd`、`bvule`、`ite` 等节点转换回题目的 `+`、`<=`、`if(cond,a,b)` 语法并提交。

不使用合成器也能构造平凡解：用嵌套 `if` 区分五组输入，再返回对应常数。题目禁止字面常量，但可从变量凑出：

```text
x-x       -> 0
x/x       -> 1（x 非 0 时）
1+1       -> 2
```

再通过加法、乘法和质因数分解构造需要的常数。这个方案受输入长度与 $x=0$ 边界影响，SyGuS 更稳定。完成 7 轮后得到形如 `flag{dIdy0uUseCvC5?-...}` 的第二段 flag。

## 方法总结

- 核心技巧：Co-Login 把反求表达式转为 36 位 SMT；Co-UnitTest 把输入输出样例转为带语法约束的 SyGuS。
- 识别信号：固定宽度溢出、表达式 AST、需要反求变量或根据有限样例生成表达式。
- 复用要点：位向量有符号/无符号运算必须匹配题目；合成结果还要做语法回译，并用五组样例在本地按 36 位语义复验。
