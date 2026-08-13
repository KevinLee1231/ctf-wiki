# GreyCTF2022 - Memory Game Part 2

## 题目简述

第二题要求在 20 秒内完成最高难度，并从 `FLAG` 标签的 logcat 读取结果。源码只在 `difficulty == 6` 且限时内获胜时解密 flag；密钥、盐和 IV 又由可复现的伪随机数生成器决定，因此既可动态绕过难度，也可离线解密。

## 解题过程

官方动态解法用 Frida hook `Engine.onEvent(FlipCardEvent)`，在事件处理前把当前 `BoardConfiguration.difficulty` 改成 6。正常完成较小棋盘后便进入解密分支：

```javascript
Java.perform(function () {
  var Engine = Java.use('com.snatik.matches.engine.Engine');
  Engine.onEvent.overload('com.snatik.matches.events.ui.FlipCardEvent')
    .implementation = function (event) {
      this.mPlayingGame.value.boardConfiguration.value.difficulty.value = 6;
      return this.onEvent(event);
    };
});
```

离线分析则更确定：`Rnd.reSeed()` 用 `com.snatik.matches` 的哈希作为 Mersenne Twister 种子，依次生成 16 字节 salt 和 16 字节 IV；口令是版本名 `1.01.001007`。密钥由 `PBKDF2WithHmacSHA1` 迭代 65536 次生成 256 位结果，再以 AES-CBC/PKCS5Padding 解开硬编码 Base64 密文。

```text
grey{hum4n_m3m0ry_i5_4lw4y5_b3tt3r_th4n_r4nd0m_4cc3ss_m3m0ry}
```

## 方法总结

运行时门槛与密码材料应分开分析。Frida 可快速满足控制流条件，而确定性 PRNG 又允许完全离线复现；两条路径相互验证。复现 Java 随机数时必须保持种子算法、`nextInt(256)` 消费顺序和有符号字节转换一致。
