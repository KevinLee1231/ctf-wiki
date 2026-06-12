# Hook My Secret

## 题目简述

移动端/so 逆向题，分为图案解锁、key 逆向和 AES 解密三层。第一层可忽略；第二层可静态分析 so 或在 Java 层 hook 加密函数做黑盒逐字节爆破；第三层通过 hook 或提取数据库拿 IV，用 key AES 解密 flag。

## 解题过程

这道题分为三层，图案解锁、key逆向、AES解密，主要想考察选手的Frida Hook能力。

第一层可以直接忽略掉，不影响解密。

第二个是so层对输入的key加密，这里可以直接静态分析so层的加密（没怎么加混淆），或者直接在java层进行黑盒分析，即hook住加密函数，输入不同的key，观察密文的变化。

如果输入111111和112111，会发现在变动的这个字符之前密文是不变的，但在2之后全都跟着变了，

因此可以直接从前到后依次爆破得到key。

第三层是用第二层输的key加密flag，通过hook，可以直接获取iv，或者手动去取出数据库文件，解析出iv，最后用AES解密得到flag。

```text
'use strict';
setImmediate(function () {
  Java.perform(function () {
    var ActivityThread = Java.use('android.app.ActivityThread');
    var ChallengeStateManager =
Java.use('com.nctf.hookmysecret.domain.ChallengeStateManager');
    var NativeBridgeClass =
Java.use('com.nctf.hookmysecret.nativebridge.NativeBridge');
    var DbHelper = Java.use('com.nctf.hookmysecret.storage.DbHelper');
    var ChallengeConfig =
Java.use('com.nctf.hookmysecret.domain.ChallengeConfig');
    var Base64 = Java.use('android.util.Base64');
    var Cipher = Java.use('javax.crypto.Cipher');
    var SecretKeySpec = Java.use('javax.crypto.spec.SecretKeySpec');
    var IvParameterSpec = Java.use('javax.crypto.spec.IvParameterSpec');
    var StringCls = Java.use('java.lang.String');
    var app = ActivityThread.currentApplication();
    if (app === null) {
      throw new Error('application is not ready');
    }
    var context = app.getApplicationContext();
    var state = ChallengeStateManager.$new(context);
    var alphabet =
'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    var keyLength = 16;
    var filler = 'A';
    console.log('\n====Hook Started====');
    function getStaticField(clazz, fieldName) {
      var field = clazz.class.getDeclaredField(fieldName);
      field.setAccessible(true);
      return field.get(null);
    }
    function getStaticStringField(clazz, fieldName) {
      var value = getStaticField(clazz, fieldName);
      if (value === null) {
        throw new Error('static field "' + fieldName + '" is null');
      }
      return value.toString();
    }
    function getNativeBridge() {
      var instance = getStaticField(NativeBridgeClass, 'INSTANCE');
      if (instance === null) {
        throw new Error('NativeBridge.INSTANCE is null');
      }
      return Java.cast(instance, NativeBridgeClass);
    }
    function getStaticIntArrayField(clazz, fieldName) {
      var value = getStaticField(clazz, fieldName);
      if (value === null) {
        throw new Error('static field "' + fieldName + '" is null');
      }
      return Java.array('int', value);
    }
    function padCandidate(prefix, ch) {
      var rest = keyLength - prefix.length - 1;
      return prefix + ch + filler.repeat(rest);
    }
    function matchingPrefixLength(candidateCipher, targetCipher) {
      var limit = Math.min(candidateCipher.length, targetCipher.length);
      for (var i = 0; i < limit; i++) {
        if (candidateCipher[i] !== targetCipher[i]) {
          return i;
        }
      }
      return limit;
    }
    function deriveAesKeyBytes(stage2Key) {
      var normalized = stage2Key.trim().replace(/-/g, '');
      return StringCls.$new(normalized).getBytes('UTF-8');
    }
    var nativeBridge = getNativeBridge();
    var targetCipher = getStaticIntArrayField(ChallengeConfig,
'stage2TargetCipher');
    var prefix = '';
    for (var pos = 0; pos < keyLength; pos++) {
      var bestChar = null;
      var bestPrefixLength = -1;
      for (var i = 0; i < alphabet.length; i++) {
        var ch = alphabet[i];
        var candidate = padCandidate(prefix, ch);
        var cipher = Java.array('int', nativeBridge.encryptStage2(candidate));
        var prefixLength = matchingPrefixLength(cipher, targetCipher);
        if (prefixLength > bestPrefixLength) {
          bestPrefixLength = prefixLength;
          bestChar = ch;
        }
      }
      if (bestChar === null) {
        throw new Error('failed to recover key at index ' + pos);
      }
      prefix += bestChar;
      console.log('Bursting ' + (pos + 1) + '/16: ' + prefix);
    }
    var finalCipher = Java.array('int', nativeBridge.encryptStage2(prefix));
    if (matchingPrefixLength(finalCipher, targetCipher) !==
targetCipher.length) {
      throw new Error('recovered stage2 key does not match target cipher: ' +
prefix);
    }
    var stage2Key = prefix;
    state.setStage1Passed(true);
    state.setStage2Passed(true);
    state.setStage2Key(stage2Key);
    var helper = DbHelper.$new(context);
    var iv = helper.getStage3Iv();
    var ivB64 = Base64.encodeToString(iv, 2).toString();
    var cipherB64 = getStaticStringField(ChallengeConfig,
'stage3CipherBase64');
    var cipherBytes = Base64.decode(cipherB64, 0);
    var keyBytes = deriveAesKeyBytes(stage2Key);
    var cipher = Cipher.getInstance('AES/CBC/PKCS5Padding');
    var keySpec = SecretKeySpec.$new(keyBytes, 'AES');
    var ivSpec = IvParameterSpec.$new(iv);
    cipher.init(2, keySpec, ivSpec);
    var plainBytes = cipher.doFinal(cipherBytes);
    var flag = StringCls.$new(plainBytes, 'UTF-8').toString();
    console.log('stage2_key=' + stage2Key);
    console.log('stage3_iv=' + ivB64);
    console.log('flag=' + flag);
  });
});
```

## 方法总结

- 核心技巧：Frida Hook 辅助黑盒分析 so 层 key 加密，并还原 AES 解密参数。
- 识别信号：输入 key 的某一位变化会导致该位之后密文全部变化时，可尝试前缀稳定的逐字符爆破。
- 复用要点：能 hook 到中间加密函数和 IV 时，优先动态取证，减少静态逆向成本。
