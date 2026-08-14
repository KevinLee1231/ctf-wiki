# Private Hidden Paths

## 题目简述

注册接口用用户可控参数拼接 PHP `pack` 格式串，签名覆盖打包后的权限和用户名。文件读取接口按权限选择 `/basic` 或 `/pro` 根目录，只阻止字面量 `../`。目标是先构造签名合法的 pro token，再借 `/proc` 路径读取容器根目录的 flag。

## 解题过程

注册函数执行：

```php
$q = pack("i$p", $a, $u);
return base64_encode(md5($q.$s).$q);
```

`$p` 完全可控。PHP `pack` 的 `X` 会把写入位置回退一个字节。令普通用户字符串开头为小端整数 `0x1337`，即字节 `37 13 00 00`，再让格式在写完普通权限 `0xb451c` 后回退并覆盖它。官方构造为：

```text
u = %37%13%00%00abcde
p = XXXXa*
```

服务端会对最终畸形数据自行计算 MD5，因此 token 签名仍然有效；验证时 `unpack("ia/a*u", $d)` 把前四字节解释为权限 `0x1337`，根目录切换为 `/pro`。

读取路径通过字符串拼接形成 `/pro` 加用户路径。提供：

```text
c/self/root/flag.txt
```

拼接结果是 `/proc/self/root/flag.txt`。它不含被过滤的 `../`，而 `/proc/self/root` 指向当前进程的文件系统根目录，所以读到容器 `/flag.txt`：

```python
token = register("%37%13%00%00abcde", "XXXXa*")
print(get_file(token, "c/self/root/flag.txt"))
```

最终得到：

```text
grey{1_l0v3_php_17_15_50_53cur3}
```

## 方法总结

这条链同时利用了二进制序列化格式注入和路径前缀拼接。签名只能证明数据由服务端生成，不能弥补生成逻辑可被操纵；路径安全也不能只过滤 `../`，应规范化后验证最终路径仍位于允许根目录。
