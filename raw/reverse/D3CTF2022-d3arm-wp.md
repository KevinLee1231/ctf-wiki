# d3arm

## 题目简述

一开始拿到文件为 bin  文件，先用 binwalk  跑一遍，并没有发现什么东西

先什么都不管，打开 ida  查询字符串列表，可以发现 stm32  和 mbed-os  这些关键词，简单查询就知道是嵌入式的文件

题目本质是 STM32/mbed-os 固件逆向：需要手动恢复 RAM 和代码段基址，按 ARMv7-M/Thumb 模式加载后定位 `main`，再通过 flag 和 key 的交叉引用还原异或解密逻辑。

## 解题过程

对于 stm32  的逆向，需要先恢复分区，一般来说，文件最开始的两个地址分别对应了芯片上电后的Ram 的起始地址和代码的开始地址，我们可以借此来恢复分区

借此可以判断 ram 的基址为 0x20000000 ，代码段的基址为 0x8000000

再次启动 ida，架构选择 arm 小端，在 详细设置里面选择 ARMv7-M

接着填入段的地址，加载好后直接跳转到起始地址 -1 的位置（0x80004C8 ）[1]，恢复代码，此时可以看到很多函数都解析出来了

搜索字符串 ‘main’ [2]，可以找到 stm32 初始化后的 main 函数，但是 ida 没有识别到，需要我们接着手动跳转恢复（这里也需要地址 - 1）

往下看，就能找到程序的主逻辑，不过这些识别上可能会有一些问题

不难找到 flag  的位置，经过分析可以发现这段的触发条件是要求 point == 42 ，模式为 hard

对 flag  交叉引用可以发现生成方式为异或，并且密文为

```c
int sub_8005E20()
{
  if ( qword_20002304 != qword_200022F4 )
    return 0;
  if ( index <= 41 )
    flag[index] = cipher[4 * index] ^ key;
  return 1;
}
```

再对密钥进行交叉引用，可以发现：

```c
_DWORD *sub_8005DB0()
{
  _DWORD *result; // r0
  int v1; // r1
  bool v2; // cc
  __int64 *v3; // r1
  __int64 *v4; // r2

  result = (_DWORD *)sub_8007850(12);
  v1 = ++index % 3;
  v2 = (unsigned int)(index % 3) > 2;
  *result = 0;
  result[1] = 0;
  result[2] = 0;
  if ( !v2 )
    key = 0x335E44u >> (8 * v1);
  v3 = &qword_20002304;
  do
  {
    v4 = v3;
    v3 = (__int64 *)*((_DWORD *)v3 + 2);
  }
  while ( v3 );
  *((_DWORD *)v4 + 2) = result;
  return result;
}
```

实际上就是一个长度为 3 的 0x335E44  逐位异或，简简单单写脚本得：

```python
def mydecodr(flag):
    cipher = ''
    key = 'D^3'
    for i in range(42):
        cipher += (chr(flag[i] ^ ord(key[i % 3])))
    return cipher

qwq = [
 0x20,      0x6D,      0x50,      0x30
,0x38,      0x48,      0x26,      0x3D
,   2,      0x75,      0x6A,         7
,0x77,      0x38,       0xA,      0x7D
,0x38,      0x56,      0x75,      0x38
,   0,      0x76,      0x3D,      0x50
,0x22,      0x3B,         5,      0x75
,0x6E,         0,      0x7D,      0x67
, 0xA,      0x71,      0x67,         3
,0x73,      0x68,         3,      0x20
,0x6E,      0x4E
]
print(mydecodr(qwq))
```

注释：

[1] 寄存器末尾最后一位是用来区别 arm 指令集和  thumb 指令集。 寄存器末尾数字为 1 表示 thumb，为 0 表示 arm 指令

[2] stm32 往往都有 ‘main’ 存在在程序中

[3] ghidra  对固件逆向要更为友好

## 方法总结

- 核心技巧：裸 `bin` 固件没有文件格式元数据，先从向量表恢复 RAM 起始地址和 reset handler，再按 STM32 常见代码基址 `0x08000000` 加载。
- 关键细节：ARM Cortex-M 使用 Thumb 模式，函数入口地址最低位为 1；在 IDA/Ghidra 跳转恢复函数时要用地址减 1 后的位置。
- 解题路径：定位 `main` 后找到 flag 触发条件 `point == 42` 和 hard 模式，再从 flag/key 交叉引用还原 `D^3` 三字节循环异或。
