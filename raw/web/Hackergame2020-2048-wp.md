# Hackergame2020 2048 WP

## 题目简述

题目把 2048 的胜利条件包装成“FLXG 大成功”。正常做法是合并出目标方块，但源码泄露了胜利后请求 flag 的接口。关键漏洞是后端只检查一个可控查询参数，没有验证玩家是否真的完成游戏。

## 解题过程

网页源代码中的 changelog 提示关注 `static/js/html_actuator.js`。胜利处理函数包含如下逻辑：

```javascript
if (won) {
  url = "/getflxg?my_favorite_fruit=" +
        ('b' + 'a' + +'a' + 'a').toLowerCase();
}
```

JavaScript 中一元加号会把第二个 `'a'` 转成 `NaN`，字符串拼接结果是 `baNaNa`，再转小写得到 `banana`。继续检查后端 `main.py`，可以确认路由只做以下比较：

```python
@app.route("/getflxg", methods=["GET"])
def getflxg():
    if request.args.get("my_favorite_fruit") == "banana":
        return hg_dynamic_flag(session["token"])
    return "还没有大成功，不能给你 flxg。"
```

因此登录题目后直接访问：

```text
/getflxg?my_favorite_fruit=banana
```

即可得到当前账号的动态 flag。这里不需要伪造游戏棋盘，也不需要执行耗时的 2048 搜索；真正的信任边界位于后端路由。

## 方法总结

前端代码不仅会暴露隐藏接口，还会暴露接口需要的参数。更重要的是，服务端不能把“只有胜利分支才会发起这个请求”当作授权依据。任何客户端请求都可被独立重放，关键状态必须在服务端保存并校验。
