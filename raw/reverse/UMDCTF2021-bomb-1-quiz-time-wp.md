# Bomb 1: Quiz Time

## 题目简述

题目给出一个 Java JAR，运行后要求回答一系列问题。所有校验逻辑和正确答案都包含在字节码中，目标是反编译 `bombOne()`，而不是逐题猜测。

## 解题过程

先列出 JAR 内容并定位主类：

```bash
jar tf bomb.jar
javap -classpath bomb.jar -c -p <MainClass>
```

也可以使用 CFR、FernFlower 等反编译器恢复接近源码的 Java。主流程中的 `bombOne()` 依次读取答案并与硬编码常量比较；沿成功分支整理常量，可以直接得到每题答案和最终字符串。

若反编译器把字符串拼接拆成多个局部变量，应按控制流顺序而不是按常量池顺序拼接。成功完成所有判断后程序输出：

```text
UMDCTF-{c00l_math_f0r_c0llege_kidZ}
```

## 方法总结

Java 字节码保留了较丰富的类型和方法信息，优先反编译通常比动态试错更快。关键是确认实际入口、成功分支和字符串构造顺序；常量池里可能同时存在提示、错误消息和 flag 片段，不能仅靠 `strings` 的出现顺序。
