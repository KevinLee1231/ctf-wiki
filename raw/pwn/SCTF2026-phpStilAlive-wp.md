# phpStilAlive

## 题目简述

服务通过网页提交并执行 PHP 8.4 代码，配置了 token 黑名单、`disable_functions`、`open_basedir` 等限制。决定性漏洞不是 HTTP 参数处理，而是 PHP `Serializable` 反序列化路径中的 `var_hash` use-after-free：用户自定义 `unserialize()` 回调内再次调用 `unserialize` 时，内外层错误共享引用表，内部对象属性表扩容会释放仍被外层 `R:N` 引用的 zval 存储。

利用 UAF 可逐步得到堆地址泄漏和任意读，再伪造 `zend_closure`，直接调用内部 `zif_system` 执行 `/readflag`。因此本题虽然放在 Web 源目录中，实际应归入 pwn。

## 解题过程

### 1. 制造 Serializable var_hash UAF

核心类非常短：

```php
class CachedData implements Serializable {
    public function serialize(): string { return '' ; }
    public function unserialize(string $data): void {
        unserialize($data)->x = 0;
    }
}
```

外层序列化串先创建带多项属性的对象，使 `var_hash` 中若干引用槽指向其属性 `arData`；进入 `CachedData::unserialize` 后再次反序列化对象并新增属性，触发 HashTable resize，旧 `arData` 被释放。由于嵌套路径没有正确隔离/锁定外层引用表，外层后续解析 `R:N` 时仍会解引用这些已释放槽。

通过调整属性数、引用编号和相邻字符串喷射，可以让释放块被可控的 zend_string 或对象数据复用。官方 `payload.php` 把该原语封装成读穿引用：第一阶段从被改写字符串长度/指针中泄漏 heap，之后伪造字符串 header，使字符串读取指向任意用户态地址。

### 2. 泄漏 PHP 运行时结构

大量喷射普通 Closure，在任意读窗口中搜索符合 `zend_object` 布局的指针对，得到 Closure 的 `ce` 与 `handlers`。再以 handlers 位于 PHP 二进制 `.bss` 附近这一关系寻找 executor globals，验证候选 HashTable 的 mask、`arData`、`nNumUsed` 等字段，最终定位：

```text
EG(function_table)
EG(symbol_table)
Closure class entry
Closure object handlers
```

每个候选都要做结构性验证，不能仅凭“像高地址”就接受，否则错误指针会在后续伪造对象时直接崩溃。

### 3. 绕过 disable_functions

`system` 被禁用后通常不再出现在当前请求的 function table 中，但标准扩展模块的静态 `zend_function_entry[]` 仍包含它对应的内部 handler。利用任意读，先从 `var_dump`、`array_push` 等仍存在的内部函数回溯所属 module，再遍历 module function entries，按函数名找到 `system`，取得真实 `zif_system` 地址。

这与重新启用 PHP 函数不同：攻击者绕过的是用户层 function table，直接让伪造内部函数对象的 handler 指向 `zif_system`。

### 4. 伪造 Closure 并执行 readflag

在可定位的全局字符串 `_xfc` 中布置伪 `zend_closure`：复制合法 Closure 的 `ce`/handlers，设置内部函数类型和参数计数，再把 handler 写成 `zif_system`。随后再次构造带 `R:N` 的反序列化 payload，把某个 zval 的类型混淆成 `IS_OBJECT`，其对象指针指向 `_xfc` 字符串数据内的伪对象。

得到可调用对象后执行：

```php
$fakeClosure('/readflag');
```

官方 `exp/payload.min.php` 是可直接提交版本，`exp/exp.py` 负责 multipart 请求和输出提取。题目来源链接中的 [Serializable UAF 分析](https://blog.calif.io/p/mad-bugs-finding-and-exploiting-a) 详细说明了共享 `var_hash`、属性表 resize 和伪 Closure 三阶段；另一条 [TimeAfterFree](https://github.com/m0x41nos/TimeAfterFree) 路线依赖 `DateInterval`，而本题已明确禁用该类，因此不应把它当成当前主解。

## 方法总结

本题的防护都位于 PHP 语言层，而利用直接破坏 Zend Engine 对象与函数结构，二者不在同一安全边界。复现应严格分成 heap leak、任意读、Closure 元数据定位、`zif_system` 解析、伪对象调用五个阶段，并使用指针范围和 HashTable 字段验证候选。外部文章只提供通用漏洞机制；实际偏移、PHP 版本差异和被禁用函数集合必须以题目附带的 PHP 8.4.22 源码、配置和官方 payload 为准。
