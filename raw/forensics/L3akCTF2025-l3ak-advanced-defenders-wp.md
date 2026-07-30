# L3akCTF 2025 L3ak Advanced Defenders Writeup

## 题目简述

附件 `Backup.dat` 的文件头为 `win-ad-ob`，它是 Sysinternals AD Explorer 保存的离线 Active Directory snapshot。把它载入 AD Explorer 后，需要从 DN、OU、计算机、用户、组和 GPO 属性中回答 14 个问题，全部正确才会返回 flag。

这题不是对在线域进行枚举；所有答案都应从离线快照中读取。

## 解题过程

### 打开快照并确定查询方法

使用 AD Explorer 的 `File -> Open Snapshot` 打开 `Backup.dat`。主要查看以下属性：

- `distinguishedName`：确定 forest 根域与对象所在 OU；
- `operatingSystem`、`operatingSystemVersion`：识别工作站系统；
- `userAccountControl`：判断禁用、密码不过期等账号状态；
- `member`、`memberOf`：检查高权限组与跨部门授权；
- `gPOptions`：判断 OU 是否阻止 GPO 继承；
- `displayName`：识别 GPO 来源与用途。

服务端比较前只做 `strip().lower()`，不会自动排序或标准化标点，因此答案顺序和逗号必须按 `build/app.py` 要求填写。

### 14 个答案及证据

| 题号 | 答案 | 证据 |
| --- | --- | --- |
| 1 | `l3ak.ctf.com` | 根 DN 为 `DC=l3ak,DC=ctf,DC=com` |
| 2 | `L3AKPRIDC` | `OU=Domain Controllers` 下的主要 DC |
| 3 | `FileSrv03, FileSrvWin11, InternStn` | 三台主机仍位于默认 `CN=Computers`，未分配 OU |
| 4 | `Windows 95, InternStn` | `InternStn` 的 `operatingSystem=Windows`、版本为 `95` |
| 5 | `ITWorkStn02, ITWorkStn03` | 两台 Windows 11 主机被放在 `OU=Windows10` |
| 6 | `IT, ITTroubleshootStn, Linux, Repo` | 位于 `CN=Deleted Objects` 的计算机对象 |
| 7 | `Wilhelm Firtz, Reginald Norwood, Christopher Price, 0x202` | 三个用户的 `userAccountControl=0x202` |
| 8 | `Bigsby Appleton, Montgomery Fitzgerald, Lily Sampson, 0x10200` | 启用且密码不过期 |
| 9 | `Finance-3, HR-8, IT-5` | `OU=Personnel` 下三个部门的启用用户数量 |
| 10 | `Charlie Edgars, Lily Sampson` | `Schema Admins` 组的成员 |
| 11 | `Christopher Price, Eleanor Wharton` | `memberOf` 指向自身部门之外的员工组 |
| 12 | `Domain Controllers, IT, FileServers` | 这些 OU 的 `gPOptions=0x1` |
| 13 | `4BD7742C73A610EDF79A6B484457351438C90DC6FAC119EF8475B46D96BD2B37` | 与 DoD GPO 对应的 `Group Policy Objects (GPOs) - April 2025` ZIP 的 SHA-256 |
| 14 | `Microsoft Defender, 7, JavaScript, VBScript` | Defender STIG GPO 的产品、定义最大天数和脚本限制 |

### 理解几个容易混淆的属性

`userAccountControl` 是位标志组合。普通账号位为 `0x200`：

$$
0x202=0x200\mathbin{\vert}0x2,
$$

其中 `0x2` 为 `ACCOUNTDISABLE`，所以题 7 的账号被禁用。

$$
0x10200=0x200\mathbin{\vert}0x10000,
$$

其中 `0x10000` 为 `DONT_EXPIRE_PASSWORD`，所以题 8 要求同时检查账号未禁用且带该位。

题 9 不能只数 OU 中的对象。Finance OU 有 4 个用户，但 Reginald Norwood 已禁用，所以 active employees 为 3。

题 12 的 `gPOptions=0x1` 表示 Block Inheritance。它影响从父容器下发的 GPO，不等于该 OU 没有自己的策略。

题 14 对应两条 Defender STIG 规则：

- `WNDF-AV-000029`：病毒定义年龄不得超过 7 天；
- `WNDF-AV-000036`：必须阻止 JavaScript 和 VBScript 启动可执行文件。

这些规则含义已经写入正文，因此无需依赖 PDF 中的外部查询链接。

### 取得 flag，并处理仓库中的拼写差异

按顺序提交 14 个答案后，当前 `build/app.py` 返回：

```text
L3AK{@ct1v3_d1r3ct0ry_3num3r@t10n_1s_fun_sdf0sa90}
```

仓库顶层 `Readme.md` 把 `d1r3ct0ry` 误写为 `d1r3t0ry`，少了一个 `c`。实际服务端常量和运行结果应作为权威值，不能照抄 README 中的拼写。

## 方法总结

AD Explorer snapshot 是可查询的证据库，不是普通文本备份。先按对象类型划分问题，再围绕 DN、`userAccountControl`、组成员关系和 `gPOptions` 等少量关键属性检索，效率远高于逐节点浏览。最后还要以服务端源码核对答案顺序和 flag；本题 README 自身存在一个字符的 flag 误写。
