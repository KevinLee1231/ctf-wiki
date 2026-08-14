# type_b_cipher

## 题目简述

题目提供一段二战时期风格的密文和提示，指向日本外务省使用的 Type B Cipher，即盟军所称的 PURPLE 密码机。题目给出的俳句和提示用于确定开关设置，最终配置为 `02 03 05 07`。

Flag 要求去掉明文中的空格后放入 `greyhats{...}`。

## 解题过程

使用 [CryptoCellar 的 PURPLE 模拟器页面](https://cryptocellar.org/simula/purple/index.html)。该页面说明 PURPLE 是日本外务省在二战前后使用的外交密码机，并提供能够处理历史报文的模拟器；旧版图形程序面向 32 位 Windows，必要时需在兼容环境中运行。

具体操作如下：

1. 启动模拟器并切换到 decipher/decrypt 模式。
2. 按顺序将四组开关设置为 `02 03 05 07`。
3. 输入附件中的完整密文，保持字母顺序不变。
4. 读取模拟器输出的英文明文，移除空格并按题目格式包装。

解出的句子为：

```text
THOSE WHO CAN IMAGINE ANYTHING CAN CREATE THE IMPOSSIBLE
```

因此 Flag 为：

```text
greyhats{THOSEWHOCANIMAGINEANYTHINGCANCREATETHEIMPOSSIBLE}
```

## 方法总结

- 核心技巧：根据历史密码机名称和提示恢复 PURPLE 模拟器的工作模式与四组开关位置。
- 识别信号：题面明确对比 Enigma 与日本外交密码、出现 PURPLE、Type B、历史报文和开关设置提示。
- 复用要点：使用历史密码模拟器时必须记录模式、初始状态和开关顺序；仅保存工具链接而不保存参数无法复现。
