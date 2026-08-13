# GreyCTF2023 CrashPython

## 题目简述

网页把用户代码交给 Judge0 的 Python 环境执行。后端不要求取得代码执行或读取文件；它只检查任务是否以信号 11 结束、消息是否为 `Exited with error status 139`，且标准错误包含 `Segmentation fault`。因此主障碍是从 Python 触发原生解释器崩溃。

## 解题过程

仓库官方解法利用 CPython 的 `ctypes` 浮点参数转换问题 CVE-2021-3177。提交以下代码：

```python
from ctypes import c_double

a = c_double.from_param(1e300)
print(a)
```

在题目部署的受影响 Python 版本中，极大的科学计数法浮点数经 `ctypes` 转换时触发底层缓冲区问题，进程以 SIGSEGV 退出。提交后轮询结果页，待 Judge0 状态从队列变为完成；后端的三项崩溃特征同时命中后返回：

```text
grey{pyth0n-cv3-2021-3177_0r_n0t?_cd924ee8df15912dd55c718685517d24}
```

某些同类环境中 `ctypes.string_at(0)` 也能直接解引用空指针，但官方脚本采用上面的版本相关触发器，复现时应以部署解释器为准。

## 方法总结

这不是普通 Web 注入题：Web 层只是转发和判定结果，真正需要越过的是 Python 与 C FFI 的内存安全边界。题目只验证崩溃特征，因此最短路径是寻找允许模块中的确定性 SIGSEGV 原语，并严格匹配判题器期望的退出码和错误文本。
