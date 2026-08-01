# Cookie Cutter

## 题目简述

服务把用户提交的 `email`、自增 `uid` 与固定的 `role=user` 序列化后，用 AES-ECB 加密并放入 Cookie。登录时服务只解密并解析字段，没有认证标签。目标是利用 ECB 分组可独立替换的性质，把角色改成 `admin`。

## 解题过程

ECB 满足相同明文块产生相同密文块，而且替换一个密文块只影响对应明文块。先构造一个完整的管理员块：让某个可控 `email` 内容在块首出现

```python
b"admin" + bytes([11]) * 11
```

其中 11 个 `0x0b` 是合法 PKCS#7 填充。官方服务在该字段前有固定序列化前缀；对比赛时常见的五位 UID，使用 10 个占位字节即可把 `admin` 推到块边界。更稳健的做法是枚举 UID 位数 $d$，令邮箱占位长度满足

$$
17+d+\text{len(email)}\equiv0\pmod{16}.
$$

取得包含独立 `admin` 块的 Cookie 后，再注册普通账户，调节邮箱长度，使 `role=` 恰好结束在分组边界。把普通 Cookie 的前两个块与管理员块拼接：

```python
forged = normal_cookie[:32] + admin_block
```

服务解密后得到 `role=admin`，访问受保护页面即可读取：

```text
byuctf{d0nt_cut_1t_l1k3_th4t!!}
```

## 方法总结

ECB cut-and-paste 的关键是先写出精确的序列化明文，再计算每个字段所在块，而不是盲切密文。即使明文被加密，只要缺少不可伪造的完整性校验，攻击者仍可重排合法分组；实际系统应使用带认证的 AEAD 模式。
