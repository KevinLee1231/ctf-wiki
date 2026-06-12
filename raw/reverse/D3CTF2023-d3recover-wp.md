# d3recover

## 题目简述

题目给出两个版本的 Cython/Python 扩展程序。`d3recover_ver2.0` 未去符号，可用 BinDiff 把符号和函数名辅助恢复到 `d3recover_ver1.0`，再定位校验函数。核心校验是对输入字节做异或、索引重排和加法/与/异或变换，最终可转成约束求解。

## 解题过程

`d3recover_ver2.0` 的符号表没有被 strip，因此可以使用 `BinDiff` 辅助恢复 `d3recover_ver1.0` 的符号表。

两个版本的 check 函数非常相似，所以在 `d3recover_ver1.0` 恢复出来的符号表中可以看到 `d3recover_ver2_check`。也可以通过 strings 窗口定位 check 函数。

重点关注以 `_Pyx` 开头的 API，很容易还原 check 算法：

```
for ( i = 0LL; i <= 31; ++i ){
    Item = (_QWORD *)_Pyx_PyInt_From_long(i);
    Item = (_QWORD *)_Pyx_PyObject_GetItem(a2, Item);
_Pyx_PyByteArray_Append(v9, v24^0x23u);
}
for ( j = 0LL; j <= 29; ++j ){
    Item = (_QWORD *)_Pyx_PyInt_From_long(j);
    v16 = (_QWORD *)_Pyx_PyInt_AddObjC(v10, qword_ABC1E8, 2LL, 0LL, 0LL);
    v17 = (_QWORD *)_Pyx_PyObject_GetItem(v9, v16);
    v16 = (_QWORD *)PyNumber_Add(Item, v17);
    v17 = (_QWORD *)_Pyx_PyInt_AndObjC(v16, qword_ABC220, 255LL, 0LL, 0LL);
    v16 = (_QWORD *)_Pyx_PyInt_XorObjC((__int64)v17, qword_ABC218, 0x54LL, 0);
    PyObject_SetItem(v9, v10, v16)
}
```

使用 `z3solver` 即可求出 flag。逻辑比较简单，也可以手工反推。

## 方法总结

- 核心技巧：未 strip 版本辅助恢复符号、BinDiff 函数匹配、识别 Cython `_Pyx_*` API、把 Python 层列表/bytearray 操作还原成约束。
- 识别信号：同题给出两个相近版本且一个保留符号时，优先做版本 diff，而不是从 stripped 版本硬逆。
- 复用要点：Cython 产物中 `_Pyx_PyObject_GetItem`、`PyObject_SetItem`、`PyNumber_*` 往往对应 Python 容器读写和算术操作，适合翻译成 z3 约束。
