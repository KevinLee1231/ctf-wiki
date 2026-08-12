# Crimewatch

## 题目简述

题目交付了两个未命名的 Android Emulator qcow2 镜像：一个是加密的 userdata，另一个是 encryption-key。设备启用了 Android file-based encryption（FBE），但没有锁屏 PIN；解开 userdata 后，需要从 TeleChat 数据库、WAL、缓存和 Android 系统残留中拼回买家、被删供应商、车辆和取货地点。关键能力是加密移动镜像与删除痕迹的证据恢复，因此归为 forensics；末尾的场景定位只是证据链中的一环。

最终四项输入顺序固定为供应商账号、车牌、买家名、四舍五入到两位小数的经纬度，交给题目 `flag.py` 校验。

## 解题过程

### 解密 Android userdata

将两个 qcow2 文件分别命名为 `userdata-qemu.img.qcow2` 与 `encryptionkey.img.qcow2`。FBE 密钥镜像足以在无 PIN 条件下解开 userdata；官方路径使用 [fbe-decrypt](https://github.com/SlugFiller/fbe-decrypt) 的 Node 脚本，输出 `userdata-decrypted.img`。该工具的关键作用是把 Android FBE metadata 和第二个镜像中的密钥材料结合，而不是把 qcow2 当作可直接挂载的 ext4。

```bash
node /path/to/fbe-decrypt/fbe-decrypt.mjs
debugfs userdata-decrypted.img
```

后续重点检查 `/data/com.grey.telechat/`、`/system_ce/0/` 与 `/media/0/`，而不是枚举整台设备的无关应用数据。

### 恢复聊天与媒体证据

实时数据库 `/data/com.grey.telechat/files/telechat.db` 的 `chats`、`messages` 表给出最近买家线程：

```text
display_name = jiawei
username     = @jiawei_pickup
message      = im here already
reply        = ok. mango, same as reserved
```

所以买家答案是 `jiawei`。

供应商会话已经从实时聊天列表删除，不能因主库中缺失就停止。下列残留将同一会话关联起来：

```text
/data/com.grey.telechat/files/telechat.db-wal
/data/com.grey.telechat/files/cache/updates/pts_000091.bin
/system_ce/0/notification_history/notification_history.xml
/system_ce/0/people/people.xml
```

它们共同给出 `Vanta Supply`、账号 `@vanta_supply`、预览文本 `same SG673... import pic attached`，以及缓存媒体引用 `cache/media/tc_4392470850.dat`。因此供应商答案为 `@vanta_supply`。

导出的缓存文件 `/data/com.grey.telechat/cache/media/tc_4392470850.dat`（另有共享副本 `/media/0/Pictures/TeleChat/IMG_20260514_164900.png`）显示完整车牌 `SG67301K`。取货图 `/media/0/Pictures/TeleChat/spot.jpg` 没有 GPS EXIF，须依据画面做轻量场景定位；画面对应 Singapore Zoo，按题目要求取两位小数后为 `1.40,103.79`。

### 验证

将恢复结果按既定参数顺序输入校验器：

```bash
python3 flag.py @vanta_supply SG67301K jiawei 1.40,103.79
```

输出：

```text
grey{tobacco_and_vaporisers_control_actdf269}
```

## 方法总结

- 核心技巧：Android FBE 解密后同时检查主库、WAL、应用缓存、通知历史和 People 数据；被删聊天通常会在这些层留下不同粒度的副本。
- 识别信号：题目同时给出 userdata 和 encryption-key qcow2、且说明无锁屏 PIN 时，应优先判断为 FBE 解密，而不是尝试直接挂载 userdata。
- 复用要点：用多份残留相互验证实体与媒体引用；当照片没有 EXIF 时，场景定位的结论必须和聊天上下文及题目所需精度一起记录。
