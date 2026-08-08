# Schrodinger's Env

## 题目简述

附件是 Android APK。Java 层只负责读取输入框，将接入码与 `AssetManager` 一起传入 JNI 导出函数 `getResultNative(ticket, getAssets())`；真正的校验和解密都在 `libchal1.so` 中。

native 层不只检查输入，还同时观察两个运行环境特征：`/proc/self/maps` 是否出现 Xposed 标记，以及 Android 系统属性 `ro.security.magic_token` 是否等于 assets 中隐藏的 token。这两个特征会进入 KDF，决定最终解密出真 flag 还是诱饵。主要工作是恢复 JNI 逻辑、资源格式与环境标签，因此归类为 Reverse，而不是单纯的 Android 组件漏洞。

## 解题过程

### 恢复接入码

JNI 先规范化用户输入：仅保留 ASCII 字母和数字，再全部转为小写。对结果计算 FNV-1a 64-bit，并与常量比较：

```text
target = 0xF625741C0FFE8C21
```

题目名本身就是提示。将 `Schrodinger's Env` 按同样规则规范化后得到：

```text
schrodingersenv
```

它的 FNV-1a 64-bit 正好等于目标常量，所以接入码可直接输入题目名，也可输入任何规范化后为 `schrodingersenv` 的字符串。校验脚本可写为：

```python
def fnv1a64(data: bytes) -> int:
    value = 0xCBF29CE484222325
    for byte in data:
        value ^= byte
        value = value * 0x100000001B3 & 0xFFFFFFFFFFFFFFFF
    return value


ticket = "".join(c.lower() for c in "Schrodinger's Env" if c.isalnum())
assert ticket == "schrodingersenv"
assert fnv1a64(ticket.encode()) == 0xF625741C0FFE8C21
```

### 恢复两个环境标签

第一个环境点来自 `/proc/self/maps`。native 在 maps 中搜索：

```text
/system/framework/XposedBridge.jar
```

它不会将整行 maps 文本送入 KDF，而是规范化为两个稳定字符串之一：

```text
命中  -> hooked:maps
未命中 -> clean:maps
```

第二个环境点来自系统属性：

```text
ro.security.magic_token
```

属性不存在、为空或不匹配时归一化为 `clean:token`，与期望 token 相等时为 `hooked:token`。

### 解码 assets 中的 token

期望 token 不在 native 代码中明文出现，而是存于 `assets/compat_profile.dat`。文件结构为：

```text
offset 0..3 : ASCII "CFG1"
offset 4    : payload length
offset 5    : XOR seed
offset 6..  : encoded payload
```

按 JNI 中的固定异或规则恢复 payload，得到：

```text
masochistic.sdk::grant_key_v1
```

这是 `ro.security.magic_token` 必须返回的精确字符串。

### 走通真实解密分支

接入码通过后，native 将 `maps_feature` 和 `token_feature` 做哈希与混合，派生 16 字节 key，并用它解密内置密文。解密结果以 `MSDK|` 开头时，程序返回前缀后的真实内容；否则进入回退链并显示：

```text
fakeflag{You_Are_Too_Clean_Bro}
```

因此正确环境组合是：

```text
ticket        = schrodingersenv
maps_feature  = hooked:maps
token_feature = hooked:token
```

可在真实 Xposed 环境中提供 maps 标记并 hook 系统属性读取，也可在离线重实现中直接向 KDF 喂入上述 canonical labels。不应只 patch 最后的成功分支，因为错误环境派生的 key 无法得到 `MSDK|` 明文。

正确结果为：

```text
miniL{hook_the_detector_not_the_branch}
```

## 方法总结

- 核心技巧：跟踪 APK 从 Java 到 JNI 的调用链，恢复输入规范化、FNV-1a 校验、assets token 解码和环境标签派生密钥的完整路径。
- 识别信号：题目名与输入哈希相呼应，同时 native 主动读取 `/proc/self/maps`、Android 系统属性和 assets 配置，说明校验绑定的是规范化环境而不仅是用户输入。
- 复用要点：优先恢复送入 KDF 的稳定标签，不必模拟整份 maps 文本；遇到解密后前缀校验时，应证明 key 派生正确，而不是粗暴改成功分支。
