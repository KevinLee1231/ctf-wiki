# canner

## 题目简述

flag.bin 是自定义序列化与分词“压缩”结果。文件由两段组成：前段是一系列词典索引，单个 NUL 字节分隔后段词典；每个整数先存字节数，再以每组 7 位、低组在前的方式保存，数据字节最高位恒为 1，从而避免前段出现 NUL。

## 解题过程

整数反序列化规则为：

~~~python
def read_int(f):
    size = f.read(1)[0]
    value = 0
    for i in range(size):
        value |= (f.read(1)[0] & 0x7f) << (7 * i)
    return value
~~~

从文件开头反复读取索引，直到 peek 到 NUL；跳过分隔符后，逐项读取“长度 + 原始 token”构成 mapping。最后按索引流连接 token：

~~~python
text = "".join(mapping[i] for i in indices)
print(text)
~~~

恢复的文本中包含：

~~~text
maple{serialization must be for translating straightforward data into complex objects is it not}
~~~

## 方法总结

分析自定义格式时应从编码器建立精确语法：字段顺序、终止条件、整数端序和位宽都要一致。源码注释对端序的措辞并不完全可靠，实际移位表达式表明低 7 位组先出现；以代码行为为准即可无损还原。
