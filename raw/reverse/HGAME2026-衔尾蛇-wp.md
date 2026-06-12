# 衔尾蛇

## 题目简述

题目附件是一个 Spring Boot JAR：入口类 `com.seal.ouroborosapp.OuroborosApplication` 从标准输入读取 token，并调用业务层 `TradeService.executeTrade(token, amount)`。表面实现是 `RiskPolicy`，但真实校验逻辑会被 `LogicSwapper + ShadowLoader` 动态替换为隐藏的 `RealRiskEngine`。`RealRiskEngine` 最终调用 `OuroborosVM.execute(token, amount)`，在自定义 VM 中校验 token。

JAR 中能直接看到的关键结构包括：

```text
BOOT-INF/classes/com/seal/ouroborosapp/OuroborosApplication.class
BOOT-INF/classes/com/seal/ouroborosapp/domain/TradeService.class
BOOT-INF/classes/com/seal/ouroborosapp/infra/LogicSwapper.class
BOOT-INF/classes/com/seal/ouroborosapp/infra/ShadowLoader.class
BOOT-INF/lib/ouroboros-api-0.0.1-SNAPSHOT.jar
```

`ShadowLoader` 会调用 `IntegrityVerifier.getDeriveKey()` 得到派生 key，再读取 `/application-data.db`，用 LCG/XOR 解出隐藏 JAR 并加载 `RealRiskEngine`。`IntegrityVerifier` 还会检查 JVM 参数，命中 `-javaagent`、`-Xdebug`、`jdwp`、`arthas`、`instrument` 等调试特征会直接退出。

## 解题过程

先确认程序调用链：入口读取输入后交给 `TradeService`，而 `TradeService` 中的 `riskEngine` 字段会被 `LogicSwapper` 替换。真实实现由 `ShadowLoader` 解密并加载。

```text
OuroborosApplication.main
  -> TradeService.executeTrade(token, amount)
  -> riskEngine.check(token, amount)
  -> RealRiskEngine.check(...)
  -> OuroborosVM.execute(token, amount)
```

`initContext()` 的核心流程如下：

1. 反射调用 `com.seal.ouroborosapi.IntegrityVerifier.getDeriveKey()` 得到派生 key。
2. 读取资源 `/application-data.db`。
3. 用 LCG 生成字节流并逐字节 XOR 解密。
4. 跳过前 128 字节，把剩余内容当作 `JarInputStream` 解析。
5. 找到并加载 `RealRiskEngine`。
6. 回到业务层执行 token 校验。

真实 VM 内置一段 `FIRMWARE` 字节数组，运行时先按派生 key 的低 8 位解密：

```java
byte keyByte = (byte) (integrityKey & 0xff);
decryptedFW[i] = (byte) (FIRMWARE[i] ^ keyByte);
```

VM 是一个栈机，内存为 `256` 个 `int`：

```text
memory[i] = token.charAt(i)    // 最多 255 字符
stack     = 计算中间值
dispatcher = 状态机分发器，混入大量 0x2000~ 垃圾状态
```

核心 opcode 映射如下：

```text
0x10 PUSH imm16       // 后跟 2 字节 big-endian 立即数
0x20 PUSH_LEN         // push token.length
0x30 LOAD             // pop addr, push memory[addr]，越界 push 0
0x35 XOR              // pop a,b, push a ^ b
0x4A JZ offset        // pop check，check == 0 时跳转
0xFF HALT             // 栈顶为 1 时返回 true
```

固件校验模板是先比长度，再逐字符比较：

```text
PUSH_LEN
PUSH <len>
XOR
JZ +1
HALT

PUSH <i>
LOAD
PUSH <ord(flag[i])>
XOR
JZ +1
HALT

PUSH 1
HALT
```

因此可以爆破 `keyByte`，把 firmware 解密后反汇编，匹配上述模板提取每个字符常量：

```python
import re
from pathlib import Path

SRC = Path("OuroborosVM.java")
bs = [
    int(x, 16)
    for x in re.findall(r"\(byte\)0x([0-9A-Fa-f]{2})", SRC.read_text())
]

def disasm(fw):
    pc, ins = 0, []
    while pc < len(fw):
        op = fw[pc]
        pc += 1
        if op == 0x10:
            val = (fw[pc] << 8) | fw[pc + 1]
            pc += 2
            ins.append(("PUSH", val))
        elif op == 0x20:
            ins.append(("PUSH_LEN",))
        elif op == 0x30:
            ins.append(("LOAD",))
        elif op == 0x35:
            ins.append(("XOR",))
        elif op == 0x4A:
            off = fw[pc]
            pc += 1
            ins.append(("JZ", off - 256 if off >= 128 else off))
        elif op == 0xFF:
            ins.append(("HALT",))
        else:
            return None
    return ins

def extract_flag(ins):
    chars = []
    for i in range(len(ins) - 5):
        if (
            ins[i][0] == "PUSH"
            and ins[i + 1] == ("LOAD",)
            and ins[i + 2][0] == "PUSH"
            and ins[i + 3] == ("XOR",)
            and ins[i + 4] == ("JZ", 1)
            and ins[i + 5] == ("HALT",)
        ):
            chars.append(chr(ins[i + 2][1]))
    return "".join(chars)

for kb in range(256):
    fw = [b ^ kb for b in bs]
    ins = disasm(fw)
    if not ins:
        continue
    token = extract_flag(ins)
    if token.startswith("flag{") and token.endswith("}"):
        print(hex(kb), token)
        break
```

另一条路线是还原 `IntegrityVerifier.getDeriveKey()`，运行时加 `-Denv=prod` 避免进入干扰状态机，并去掉调试参数。`ouroboros.key` 可触发 `STABLE_RAW_KEY` 打印，辅助定位稳定 key。

## 方法总结

- 核心技巧：从 Spring Boot 动态类加载链定位隐藏校验类，再把 VM 固件解密并反汇编成逐字符比较。
- 识别信号：JAR 中出现 `ShadowLoader`、动态加载隐藏 JAR、`IntegrityVerifier`、自定义 VM firmware 时，重点看资源解密和 VM opcode。
- 复用要点：遇到反调试 JVM 参数检查时，先静态分析加载链；VM 题优先恢复 opcode 和校验模板，不必完整还原所有垃圾状态。
