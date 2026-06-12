# No Name

## 题目简述

本题是 Android/Java 方向的逆向题，核心是运行时反射替换验证接口。真实校验代码被 AES 加密放在 `assets` 中，运行时通过 native 函数取密钥解密，再执行简单异或校验。

题目附件是 Android/Java + native 的混合逆向题，assets 中的 Java 校验代码被 AES 加密，native 层负责生成或返回解密 key。native 层虽然存在反调试和防 patch 逻辑，但它主要保护取 key 的过程；更直接的做法是写一个 app 调用 native 取 key 函数，解密 assets 中的校验代码。

## 解题过程

题目源码见 [`agfn/2021-d3ctf-noname`](https://github.com/agfn/2021-d3ctf-noname)。下文保留了仓库中最关键的运行时加载关系：assets 内 AES 加密的 Java 校验代码、native 取 key、反射替换验证接口。

题目使用 Java 反射特性，在运行时把用于混淆的验证接口替换为真实验证代码。验证相关代码通过 AES 加密存放在 assets 里，运行时从 native 中获取密钥解密。native 中存在反调试和防 patch 逻辑，但更直接的做法是写一个最小 app 调用取 key 函数，解密 assets 中的真实校验代码；解密后逻辑很简单，主要是异或校验。

## 方法总结

- 核心技巧：针对 Java 反射加载和 assets 加密代码，优先恢复运行时真实验证逻辑，而不是被接口层混淆牵着走。
- 识别信号：验证接口在运行时被替换、assets 中存在加密代码、native 层只负责返回 AES key 或执行少量保护逻辑。
- 复用要点：native 反调试不一定是主障碍；如果目标函数能被正常调用，可以用最小 app 或脚本复用原函数取 key，再静态分析解密后的 Java 校验代码。

