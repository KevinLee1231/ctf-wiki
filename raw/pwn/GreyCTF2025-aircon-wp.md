# Aircon

## 题目简述

程序维护 10 台空调的遥控器温度和实际温度，要求把所有实际温度设为 25；但每次修改后，所有遥控器显示温度又必须两两不同。漏洞来自用 `%d` 向 16 位局部变量写入 32 位整数，覆盖相邻栈变量。

## 解题过程

`change_aircon_temp` 的局部变量依次包含 `remote_id`、`aircon_id` 和 `temperature`，但输入使用：

```c
scanf("%d", &temperature);
```

`%d` 会写 4 字节，而 `temperature` 只有 2 字节，因此高 16 位可以覆盖邻近的 `aircon_id`。校验函数只把低 16 位解释为温度；固定选择 remote 5，它原本显示 25，因此不会改变遥控器数组的唯一性。对实际空调编号 $i$，输入 32 位值：

$$
v=(i\ll16)\mathbin{|}25.
$$

例如自动生成交互：

```python
for i in range(10):
    print(1)                  # 修改
    print(5)                  # remote_id
    print((i << 16) | 25)    # high16=aircon_id, low16=temp
print(3)
```

每轮都把 `AIRCON_REMOTE_TEMP[5]` 保持为 25，却把不同的 `AIRCON_ACTUAL_TEMP[i]` 更新为 25。最后选择获取 flag：

```text
grey{one_rem0te_controls_a11_the_air_conditioners!}
```

## 方法总结

格式化输入必须与目标变量类型严格匹配，16 位整数应使用 `%hd`。这里的栈覆盖不是简单改返回地址，而是精准修改相邻逻辑变量；同时利用固定值 25 绕过“遥控器显示互异”检查。分析这类漏洞时，应按小端序拆分 32 位输入，明确低半字和高半字分别落到哪个局部变量。
