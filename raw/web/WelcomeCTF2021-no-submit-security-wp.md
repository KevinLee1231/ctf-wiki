# No Submit Security

## 题目简述

WelcomeCTF2021 的 No Submit Security 在 session 中生成 100 组随机键值。页面把这些键值全部输出为隐藏表单字段，服务端要求 POST 数据与 session 中每一项一致，但页面故意不显示提交按钮。

## 解题过程

服务端检查逻辑为：

```php
foreach ($_SESSION as $key => $value) {
    if (isset($_POST[$key]) && $_POST[$key] == $value) {
        continue;
    }
    $flag = false;
    break;
}
```

同一页面随后又把所有 session 键值直接写进隐藏输入框：

```html
<form id="1" method="POST" action="./index.php">
    <input type="hidden" name="..." value="...">
    <!-- 其余字段结构相同 -->
</form>
```

因此客户端已经拥有服务端所需的全部数据，缺少的只是 UI 按钮。在浏览器控制台直接提交现有表单：

```javascript
document.getElementById("1").submit();
```

也可以在开发者工具中给表单添加 `<button type="submit">Submit</button>`。浏览器会自动提交全部隐藏字段，服务端逐项检查成功后输出：

```text
greyhats{5U8m1551O5_15_FRoM_tH3_CL13Nt_51d3}
```

## 方法总结

隐藏字段和缺失按钮都不是安全边界。任何发送到客户端的数据都可被用户读取、修改和重新提交；真正的授权信息必须只保存在服务端，并基于不可由客户端伪造的状态判断。
