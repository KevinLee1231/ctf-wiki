# baby-fmtstr

## 题目简述

程序让用户选择 locale 和 `strftime` 格式，把最多 0x30 字节的格式化结果复制到 0x20 字节全局数组 `output`。紧随其后的全局数组 `command` 初始为 `ls`，退出时会执行 `system(command)`。虽然格式字符串被限制为偶数下标必须是 `%`，仍可借助 locale 相关的日期文本制造精确溢出。

## 解题过程

漏洞点是无边界的 `memcpy`：

```c
strftime(buf, 0x30, input, time_struct);
buf[strlen(buf) - 1] = '\0';
memcpy(output, buf, strlen(buf));
```

`output` 与 `command` 在 `.bss` 中相邻。目标是让格式化结果的前 32 字节填满 `output`，后两个字节恰好是 `sh`。比赛时选择 `xh_ZA.utf8`，十二月的 `%b` 会输出 `Tsh`；于是让字符 `T` 落在 `output[31]`，`s`、`h` 覆盖 `command[0:2]`，原本的 NUL 结尾保持不变。

`%G` 输出四位 ISO 周数年份，`%%` 输出一个字面量 `%`。官方脚本按输出长度动态构造：

```python
locale_name = b"xh_ZA.utf8"
month_text = b"Tsh"
size = 0x20 - len(month_text) + 2
years = size // 4
fmt = b"%G" * years + b"%%" * (size - years * 4) + b"%b"
```

最终格式为七个 `%G`、三个 `%%` 和 `%b`。发送行末换行也会进入 `strftime` 输出，随后错误的“去换行”语句删掉最后一字节，不影响前面的精确布局。选择退出后，`goodbye()` 调用 `system("sh")`，读取：

```text
grey{17'5_b0f_71m3}
```

## 方法总结

这不是 `printf` 型格式化字符串写原语，而是 `strftime` 可变长展开引发的全局缓冲区溢出。locale 会改变 `%a`、`%b` 等格式项的具体文本，因此也是攻击面的一部分。复现历史题时还要注意月份依赖；若当前日期不同，应重新搜索可用 locale 与格式组合，而不是照抄十二月的载荷。
