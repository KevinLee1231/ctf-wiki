# POKéPTCHA

## 题目简述

页面要求识别一张宝可梦剪影，三个预设答案都会失败，真正的校验函数 `run()` 藏在高度混淆的 `captcha.js` 中。脚本用自定义栈式虚拟机执行校验，还加入持续触发 `debugger` 的反调试代码。

题目没有服务端漏洞；决定性障碍是恢复 JavaScript 虚拟机与加密校验逻辑，因此归入逆向方向。

![宝可梦动画中的圆形黑色剪影；该视角对应“从上方看到的胖丁”这一经典谜题](UMDCTF2023-pokeptcha-wp/pokemon-from-above-silhouette.png)

## 解题过程

入口首先每 10 ms 执行一次 `debugger`：

```javascript
setInterval(function () {
  eval("debugger");
}, 10);
```

离线分析时可在隔离的 JavaScript 上下文中把 `setInterval` 替换为空函数。不要修改原函数 `FQgZw` 的正文，因为后续密钥会用到 `FQgZw.toString()`；插入日志也会改变密钥。

变量 `HbBiul` 是第一层字节码。解码方式为：

```javascript
const bytecode = atob(HbBiul)
  .split("|")
  .map(Number)
  .map((x) => x ^ 21);
```

VM 使用数组作为栈。初始操作码 0 负责压入立即数或字符串，1 对栈顶字符串执行 `eval`，2 把动态生成的处理函数写入操作码表；后续算术、取属性、调用和跳转处理器也都以字符串形式装入。恢复这些处理器后，末端校验可以化简为：

```text
key = MD5(location.hostname + FQgZw.toString())
ciphertext = RC4(key, answer)
Base64(ciphertext) == expected
```

原比赛域名为 `pokeptcha.chall.lol`。在不改动原函数文本的环境中，密钥和从字节码比较序列还原的目标值分别是：

```text
key      = bc493a282bbecd7339515be0667610a6
expected = g5+Kqs+Smi1f6zGOZq423PAHr0mk3LCn2vF+TdTWNH+uJ98Wt5iNLyaB
```

RC4 加解密相同，因此对目标值做 Base64 解码后再用该密钥跑一次 RC4，即可恢复输入：

```javascript
function rc4(key, data) {
  const s = Array.from({ length: 256 }, (_, i) => i);
  let j = 0;
  for (let i = 0; i < 256; i++) {
    j = (j + s[i] + key.charCodeAt(i % key.length)) & 255;
    [s[i], s[j]] = [s[j], s[i]];
  }

  let i = 0;
  j = 0;
  let out = "";
  for (const ch of data) {
    i = (i + 1) & 255;
    j = (j + s[i]) & 255;
    [s[i], s[j]] = [s[j], s[i]];
    out += String.fromCharCode(
      ch.charCodeAt(0) ^ s[(s[i] + s[j]) & 255]
    );
  }
  return out;
}

console.log(rc4(
  "bc493a282bbecd7339515be0667610a6",
  atob("g5+Kqs+Smi1f6zGOZq423PAHr0mk3LCn2vF+TdTWNH+uJ98Wt5iNLyaB")
));
```

输出：

```text
Th3_4n5W3R_1s_A_p1gGlyJuFf_S3en_fR0m_480Ve
```

将其填入输入框，`run()` 返回 `true`。完整 flag 为：

```text
UMDCTF{Th3_4n5W3R_1s_A_p1gGlyJuFf_S3en_fR0m_480Ve}
```

## 方法总结

- 图片提示来自经典的“从上方看到的胖丁”桥段，但只能确定语义，无法猜出精确大小写和 leetspeak；最终答案必须从校验逻辑恢复。
- 第一层 Base64 和异或只是字节码封装，真正逻辑由运行时 `eval` 生成的 VM 处理器完成。
- 反调试环境应在外围替换定时器，不能改动参与 `toString()` 派生密钥的函数正文。
- 识别出 MD5、RC4 和 Base64 的组合后无需暴力搜索；RC4 的对称性允许直接解密固定比较值。
