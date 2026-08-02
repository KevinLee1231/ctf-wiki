# N1CTF 2022 - n1oj_warmup

## 题目简述

题目要求在受限的 Lua 5.4 在线判题环境中读取两个高精度十进制整数并输出它们的和。环境没有任意模块加载、文件访问或外部命令能力，只保留了 `gets`、`print` 以及少量字符串和表操作。

决定解法的障碍是受限语言运行环境：不能依赖大整数库，也不能把输入直接交给系统工具，只能用暴露的基础原语自行实现十进制加法。

## 解题过程

### 确认可用原语

官方题解材料中通过枚举 `_G` 确认环境为 Lua 5.4，可用的输入输出主要是：

```text
gets   -- 读取一行
print  -- 输出并换行
```

同时还能使用字符串取长度、取字节、切片，以及表的基本操作。这些能力已经足够实现逐位加法。题解材料中的终端截图只包含上述文字信息，因此直接转写，不保留截图。

### 用低位在前的数组表示大整数

把十进制字符串从右向左转换为数字数组，使个位放在下标 1。这样进位方向与数组遍历方向一致：

```lua
local function to_digits(s)
    local out = {}
    for i = #s, 1, -1 do
        out[#out + 1] = s:byte(i) - string.byte("0")
    end
    return out
end
```

对两个数组逐位相加，并显式维护进位：

```lua
local function add_digits(a, b)
    local n = #a
    if #b > n then
        n = #b
    end

    local out = {}
    local carry = 0
    for i = 1, n do
        local total = (a[i] or 0) + (b[i] or 0) + carry
        out[i] = total % 10
        carry = total // 10
    end
    if carry ~= 0 then
        out[n + 1] = carry
    end
    return out
end
```

最后逆序拼接数字。完整提交脚本如下：

```lua
local function getline()
    local s = gets()
    if s:sub(-1) == "\n" then
        s = s:sub(1, -2)
    end
    return s
end

local function to_digits(s)
    local out = {}
    for i = #s, 1, -1 do
        out[#out + 1] = s:byte(i) - string.byte("0")
    end
    return out
end

local function add_digits(a, b)
    local n = #a
    if #b > n then
        n = #b
    end

    local out = {}
    local carry = 0
    for i = 1, n do
        local total = (a[i] or 0) + (b[i] or 0) + carry
        out[i] = total % 10
        carry = total // 10
    end
    if carry ~= 0 then
        out[n + 1] = carry
    end
    return out
end

local function from_digits(a)
    if #a == 0 then
        return "0"
    end
    local s = ""
    for i = #a, 1, -1 do
        s = s .. string.char(a[i] + string.byte("0"))
    end
    return s
end

local a = to_digits(getline())
local b = to_digits(getline())
print(from_digits(add_digits(a, b)))
```

仓库中的官方 `sol.lua` 采用同一思路，只是预先把每个数组补到 5000 位，再固定循环 5000 次。

## 方法总结

语言沙箱题首先要盘点真实可用的全局对象，而不是假设标准库完整存在。本题没有可利用的逃逸原语，预期做法是在最小 I/O 和字符串操作集合上实现高精度加法。使用低位在前的数组可把每一位的计算统一为 $s_i=a_i+b_i+c_i$、$c_{i+1}=\lfloor s_i/10\rfloor$，无需依赖宿主语言的整数精度。
