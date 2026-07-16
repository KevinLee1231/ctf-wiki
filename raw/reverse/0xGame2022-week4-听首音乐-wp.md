# week4听首音乐

## 题目简述

附件是一段可以正常播放的 MIDI 音乐，但音符同时构成了 Velato 程序。目标是识别这种以音高关系编码指令的深奥编程语言，执行程序并还原其输出。

## 解题过程

Velato 使用 MIDI 音符承载程序：首个音符建立基准音，后续音符相对当前基准音的音程表示不同指令，部分指令还会改变新的基准音。因此这类文件不能只按音频隐写检查，还应把音符序列视为可执行代码。使用 Velato 编译器读取 MIDI 后，可以把音符程序转换为普通可执行程序。

编译器的关键输出显示附件有两个音轨，它选择第一个包含音符信息的音轨，并将其解析为重复的字符输出指令：

~~~text
vlt.exe music.mid
2 tracks found, will read 1st track containing note information.
Program
    DeclareFunction
        PrintToScreen
            CharConstant
        PrintToScreen
            CharConstant
        ...
~~~

运行编译结果，程序只输出一段提示和一个十进制大整数：

~~~text
What a long number:4642488275724448709921860001805542920743247922240305533
~~~

这个整数按大端序解释为一串字节即可，无需猜测额外加密算法。可直接用 Python 整数接口完成转换：

~~~python
value = 4642488275724448709921860001805542920743247922240305533
length = (value.bit_length() + 7) // 8
plaintext = value.to_bytes(length, byteorder="big")
print(plaintext.decode())
~~~

输出为：

~~~text
0xGame{StRAnGe_eSOL4N9}
~~~

## 方法总结

MIDI 不仅能保存可听见的旋律，也能作为编程语言的语法载体。识别 Velato 后，关键步骤是按音程语义编译或解释音符；程序输出的大整数则应优先尝试按大端整数直接转换为字节，因为 flag 的 ASCII 编码本身就可以视为一个长整数。
