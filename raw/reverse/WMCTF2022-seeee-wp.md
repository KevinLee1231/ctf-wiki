# seeee

## 题目简述

题目附件是命令行程序 `seeee.exe`。程序需要两个输入：第一个类似 IV/置换种子，第二个是待校验的密文/密码文本。校验分两层：先通过一组字符映射关系把输入变换到比较数据，再进入 ChaCha20 风格的流密码处理；由于流密码加解密同构，可以把待比较数据放回相同流程中反推出期望输入。

## 解题过程

该问题需要两个输入，即加密的IV和密码文本。

第一个输出数据是输入，在二进制树之间传递。通过输入的结果，与这里的比较可以得到输入。 可以得到正确的输入。

```
ABCDEFGHIJKLMNOPQRSTUVWX  input  
KVBJIQHGEPOFUWTNADCSLXRM  output

h7BOJpTCYsuoAUQn6qFxXyVE  The same mapping, get the correct input
uy7sY6CTJnQpXVxUhOBFoEqA  data to compare
```

下面是chacha20的流密码加密，加密过程也就是解密过程。将待比较的数据放入读取数据的存储器中，操作完成后，可以得到预期的输入数据。

```
0x80, 0x1F, 0x94, 0xB4, 0xEF, 0xD4, 0x9C, 0x36, 0x47,0x85,0xE7, 0x26, 0x64, 0x4B, 0x29, 0x95, 0x1E, 0x0D, 0x39, 0xA9,0x1E, 0x72, 0x7A, 0x1F, 0xB0, 0x48, 0x22, 0x1E, 0x8E, 0x40,0xEB, 0xBF, 0x75, 0x17, 0x16, 0xD3, 0x39, 0x4F, 0xFD, 0x0A,0x58, 0x39, 0x4C, 0x9C, 0x13, 0x41, 0x8B, 0x93, 0xB2, 0x84, 0xE7, 0x2F, 0x03, 0xD4, 0x62, 0x44, 0xFC, 0x9D, 0x76, 0xEF, 0x0F, 0xF8, 0x06
```

调整输入为

```
"D:\rustwork\seeee\target\release\seeee.exe" h7BOJpTCYsuoAUQn6qFxXyVE ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+*
```

在适当的地方用比较的数据替代输入，最后在比较时得到正确的输入。

```
PqlCkyAhnbe4DKs7NE2Oz_Ycif9pQt5uLgZXvWSFmGrx8JaVj61BMHR3dUowTI0
```

将获得的输入进行两次串联。

```
seeee.exe h7BOJpTCYsuoAUQn6qFxXyVE PqlCkyAhnbe4DKs7NE2Oz_Ycif9pQt5uLgZXvWSFmGrx8JaVj61BMHR3dUowTI0
wow~ you win！
wmctf{PqlCkyAhnbe4DKs7NE2Oz_Ycif9pQt5uLgZXvWSFmGrx8JaVj61BMHR3dUowTI0}
```

## 方法总结

- 核心技巧：先恢复字符置换表，再利用 ChaCha20 流密码“加密即解密”的性质反推第二段输入。
- 识别信号：程序要求 IV 和 ciphertext 两个参数，内部出现固定映射表和一段字节流常量时，应检查是否是置换层叠流密码层。
- 复用要点：这类题不一定要完整重写整个程序，只要定位比较数据、输入映射和流密码调用点，就可以把目标比较值作为输入跑逆向等价流程；最终验证必须用原程序重新运行，确认输出 `wow~ you win!` 和 flag。
