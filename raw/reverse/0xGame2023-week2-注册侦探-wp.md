# 注册侦探

## 题目简述

程序从当前用户注册表读取 `HKEY_CURRENT_USER\Software\0xGame` 下的 `registered` 值。只有该值作为 32 位整数等于 1 时，才把内置数据逐字节与 `0x33` 异或并输出 flag。

## 解题过程

IDA 中可以看到 `RegOpenKeyExA` 打开 `SOFTWARE\0xGame`，`RegQueryValueExA` 查询 `registered`，随后比较读取的 DWORD 是否为 1。省略错误处理后的关键调用关系如下：

```c
RegOpenKeyExA(HKEY_CURRENT_USER, "SOFTWARE\\0xGame", 0, KEY_READ, &key);
RegQueryValueExA(key, "registered", NULL, NULL, (BYTE *)&data, &size);
if (data == 1) {
    print_decrypted_flag();
}
```

在 PowerShell 中创建键并写入 DWORD 值：

```powershell
New-Item -Path "HKCU:/Software/0xGame" -Force | Out-Null
New-ItemProperty `
  -Path "HKCU:/Software/0xGame" `
  -Name "registered" `
  -PropertyType DWord `
  -Value 1 `
  -Force | Out-Null
```

重新启动题目程序，注册表判断成立，程序执行异或解码并显示 flag。这里必须使用 `HKCU`，因为程序传给 `RegOpenKeyExA` 的根键是 `HKEY_CURRENT_USER`；写到 `HKLM` 或创建字符串类型值都不符合判断。

## 方法总结

Windows 程序常把授权状态保存在注册表。分析时应记录根键、子键、值名、数据类型和比较值五项信息，并注意 32/64 位注册表视图差异。客户端可写的注册表值不能作为安全授权凭据，最多只能保存普通偏好设置。
