# bank-1

## 题目简述

服务使用 AES-ECB 加密转账请求。管理员请求中含有未知的 6 位 PIN，而普通用户可以控制收款人编号并取得对应密文。由于 ECB 对相同明文分组产生相同密文分组，可以通过调整可控字段的长度，把未知 PIN 的下一个字符推到目标分组末尾，再用候选字符制造相同分组。

## 解题过程

先按源码确认请求序列化格式，计算固定前缀、recipient_id 与 PIN 在明文中的位置。每一轮选择一个长度合适的收款人编号，使管理员请求中“已恢复 PIN + 下一位”恰好落在一个完整的 16 字节块内。

随后对数字 0 至 9 分别构造普通转账请求，将同样的已知前缀、已恢复字符和候选数字对齐到同一块。如果候选请求的某个密文块与管理员请求目标块相等，就说明该候选数字正确：

~~~python
pin = ""
for pos in range(6):
    admin_ct = get_admin_money_request(alignment_for(pos))
    target = block(admin_ct, target_block_for(pos))

    for digit in "0123456789":
        probe = build_probe(pin + digit, alignment_for(pos))
        probe_ct = send_money(probe)
        if block(probe_ct, probe_block_for(pos)) == target:
            pin += digit
            break
~~~

恢复完整 PIN 后，以管理员身份把 100000 单位资金转入自己的用户，再购买特殊访问权限，服务返回：

~~~text
maple{4LL_Y0UR_m0NEy_4Re_8eL0Ng_70_u2}
~~~

## 方法总结

这是典型的 ECB 字节级匹配攻击。关键不是“解密 AES”，而是精确重建序列化文本和分组边界，使未知字符能够由密文块相等性判定。若服务允许攻击者同时取得含秘密的密文和可控明文的密文，即使密钥完全随机，ECB 仍会泄漏结构。
