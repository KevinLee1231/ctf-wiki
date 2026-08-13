# GreyCTF2022 - Manipulation

## 题目简述

转账函数按值接收一个封装 `FILE *` 的 logger。临时副本在函数结束时析构并关闭文件，但原对象仍保留同一指针，形成针对 glibc `FILE` 结构的 UAF。随后账户数组扩容可复用这块内存并控制其字段。

## 解题过程

先触发一次 transfer，让按值复制的 logger 析构，原 logger 指向已释放的 `FILE` chunk。通过安排账户对象和字符串分配，使其重占同尺寸 chunk；账户中的浮点/字符串字段既可泄露堆与 libc 指针，也可逐步覆盖旧 `FILE`。

利用脚本根据泄露计算 libc 基址，重建 `_lock` 等必须有效的字段，并伪造宽字符/虚表相关指针，使下一次 `fclose` 沿受控调用链跳到 one-gadget。

```python
transfer(...)                 # 临时 logger 析构，留下 FILE UAF
resize_accounts()             # 重占释放的 FILE chunk
libc.address = leak_libc() - known_offset

fake_file = build_file(
    lock=writable_lock,
    vtable=fake_vtable,
    target=libc.address + one_gadget_offset,
)
overwrite_dangling_file(fake_file)
trigger_logger_destructor()
```

成功后得到：

```text
grey{just_u5e_rU5t_lM40_rt56a}
```

## 方法总结

拥有资源的 C++ 包装类若使用默认浅拷贝，会造成 double-close 或 UAF。`FILE` 利用不能只改一个函数指针：现代 glibc 会检查虚表，且 `_lock` 等字段会提前被解引用，因此应按实际 `fclose` 调用路径构造最小一致对象。
