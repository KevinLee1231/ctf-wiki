# gun

## 题目简述

题目附件是经过梆梆免费版二代壳保护的 Android 应用。应用把 flag 的各个字符拆到多个线程中，每个线程休眠不同时间后向服务器发送数据；线程的休眠时间就是“子弹”的发射顺序。因此决定性工作不是破解网络协议，而是脱壳后恢复线程中的字符与顺序映射。官方题解还给出了一条抓包路线：绕过 SSL Pinning 后，按请求出现的先后顺序重组字符。

## 解题过程

先用 `frida-dexdump` 从运行中的应用内存导出 DEX。二代壳的主要保护目标是磁盘中的原始 DEX，代码加载后仍需以可执行形式存在于内存，因此内存 dump 可以取得实际业务代码。将导出的 DEX 交给 JADX，并导出 Java 源文件。

业务代码会启动大量继承 `Thread` 的类。真正参与题目的类同时包含固定的证书公钥摘要：

```text
sha256/ocfaPpOi8wBS01tMzoT6f+q+zF7ufbbxSe2wQUcpqXY=
```

每个目标类中可以提取两个量：`String str2 = "x"` 保存本线程代表的字符，向 `https://hgame.vidar.club` 发起请求时使用的数值参数对应休眠时间。个别类没有显式字符，此时以类名作为字符。下面是按官方思路整理的核心脚本；`source_root` 应改为 JADX 导出目录：

```python
import re
from pathlib import Path

source_root = Path(r"D:\CTFs\gun\src\main\java\p000")
pin = "sha256/ocfaPpOi8wBS01tMzoT6f+q+zF7ufbbxSe2wQUcpqXY="
bullet_pattern = re.compile(r'String str2 = "(.{1})"')
time_pattern = re.compile(r'https://hgame\.vidar\.club", j, (.+),')

bullets = {}
for java_file in source_root.rglob("*.java"):
    text = java_file.read_text(encoding="utf-8", errors="ignore")
    if "extends Thread" not in text or pin not in text:
        continue

    time_match = time_pattern.search(text)
    if time_match is None:
        continue

    char_match = bullet_pattern.search(text)
    char = char_match.group(1) if char_match else java_file.stem
    delay = int(time_match.group(1).strip().rstrip("Ll"))
    bullets[delay] = char

flag = "".join(char for _, char in sorted(bullets.items()))
print(flag)
```

排序的依据必须是延时，而不是文件名或 JADX 的输出顺序。这样按“发射时间”排列所有子弹，即可恢复服务端收到的 flag 字符序列。官方 PDF 说明了完整恢复方法，但没有保存静态 flag。

另一条路线是动态抓包。应用启用了 SSL Pinning，直接安装代理证书仍会握手失败；可在受控测试设备上用 TrustMeAlready 等 Hook 模块禁用证书绑定，再捕获各线程发出的 HTTPS 请求。网络到达顺序同样由线程休眠时间决定，按时间重组请求中的字符即可。该路线的关键仍是恢复顺序，不是绕过服务器鉴权。

## 方法总结

这道题把 flag 拆成“字符载荷”和“时间顺序”两部分。静态路线需要先从内存恢复被壳隐藏的 DEX，再筛选真正的线程类并按 sleep 值排序；动态路线则绕过 SSL Pinning 后从请求时序还原相同信息。APK、HTTPS 和证书绑定只是载体，核心障碍是恢复应用代码及其线程语义，所以归入 Reverse。
