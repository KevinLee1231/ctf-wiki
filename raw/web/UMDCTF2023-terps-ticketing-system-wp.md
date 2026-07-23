# Terps Ticketing System

## 题目简述

题目模拟 UMDCTF 门票系统。正常提交表单后，服务器随机生成 1 到 1000 的票号并跳转到 `/ticket?num=<票号>`。

票号完全由客户端查询参数决定，而特殊票号 0 会进入 flag 模板。

## 解题过程

正常请求得到类似跳转：

```http
Location: /ticket?num=637
```

修改地址栏中的 `num` 可以访问任意编号。后端关键判断为：

```python
num = request.args["num"]
if num == "0":
    return render_template("flag.html")
```

随机数生成范围是 `random.randrange(1, 1001, 1)`，所以合法流程永远不会产生 0，但路由并没有验证票号是否由服务器签发。直接访问：

```text
/ticket?num=0
```

页面返回：

```text
UMDCTF{d0nt_b3_@n_id0r_@lw@ys_s3cur3_ur_tick3ts}
```

## 方法总结

- 这是典型的 IDOR/业务逻辑缺陷：对象标识由用户控制，服务端没有做授权或签发状态校验。
- “正常界面不会生成某个值”不等于后端不可达，边界值 0 应主动测试。
- 修复时应把票号与服务端会话或数据库记录绑定，并验证当前用户是否拥有该票，而不是只比较查询字符串。
