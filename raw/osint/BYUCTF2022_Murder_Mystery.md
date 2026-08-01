# BYUCTF 2022 - Murder Mystery

## 题目简述

题面给出一串二进制和日期 `June 29, 1902`，要求找出与之关联的著名墓志铭，并将空格改为下划线提交。

## 解题过程

把位串每 8 位分组并按 ASCII 解码：

```python
bits = "0110111001110010..."
print(bytes(int(bits[i:i+8], 2) for i in range(0, len(bits), 8)).decode())
```

结果为：

```text
nrhpid 80002319
```

`NRHP ID` 是美国 National Register of Historic Places 的登记编号。查询 `80002319` 可定位到 Jesse James Birthplace。日期 1902-06-29 则是 Jesse James 遗体从第一处墓地迁出、与妻子合葬前被挖掘的日期，因而应查询他第一座墓碑上的铭文。

该铭文为：

```text
Murdered by a traitor and coward whose name is not worthy to appear here
```

规范化后提交：

```text
byuctf{murdered_by_a_traitor_and_coward_whose_name_is_not_worthy_to_appear_here}
```

## 方法总结

二进制只提供检索键，日期负责确认具体墓碑版本。历史人物可能有迁葬和多块墓碑，必须用日期把“著名铭文”约束到第一处墓地，不能只凭人物姓名抄任意引文。
