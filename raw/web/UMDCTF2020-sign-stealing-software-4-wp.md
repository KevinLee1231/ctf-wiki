# Houston Astros' Sign Stealing Software - Part 4

## 题目简述

页面实现了完全位于浏览器端的棒球小游戏。混淆 JavaScript 在开始 10 秒后检查得分是否至少为 `0x2710`，即 10000，并要求少于三次好球；正常交互不可能在时限内达到该分数。

## 解题过程

沿 `#start` 点击事件找到 `setTimeout(..., 0x2710)`。回调中的胜利条件为：

```javascript
runs >= 0x2710 && !lost && strikes < 3
```

可以在本地脚本中把比较门槛从 `0x2710` 改成 0，或在断点处把闭包变量 `runs` 改为 10000，再继续执行。胜利分支会显示 `#winDiv`，并把一段混淆字符串写入 `#showScore`。

脚本使用 RC4 字符串表和数组轮转。按包装器参数 `0xd5` 还原轮转后，解码索引 `0x7f`、密钥 `g&ck`，得到：

```html
<p>UMDCTF-{H0Ust0N_L0sT_4t_Th31r_@wN_G@m3}</p>
```

因此 flag 为：

```text
UMDCTF-{H0Ust0N_L0sT_4t_Th31r_@wN_G@m3}
```

其 SHA-256 与 README 完全一致。

## 方法总结

客户端游戏的计分、胜负判断和奖励文本都可被调试或修改，不能承担安全验证。处理字符串混淆时，应先定位真正被胜利分支引用的索引，只解必要项，避免在反调试垃圾代码上浪费时间。
